
conventionalChangelogConventionalCommitsVersion = 'v4.5.0'
semanticReleaseVersion = 'v17.3.1'

def getRepo() {

	checkout([$class: 'GitSCM',
		branches: scm.branches,
		browser: scm.browser,
		doGenerateSubmoduleConfigurations: false,
		extensions: [
			[$class: 'RelativeTargetDirectory', relativeTargetDir: 'repo'],
			[$class: 'PruneStaleBranch'],
			[$class: 'CleanCheckout']
		],
		submoduleCfg: scm.submoduleCfg,
		userRemoteConfigs: scm.userRemoteConfigs
	]).GIT_COMMIT
}

def getNextSemanticReleaseVersion(semanticReleaseOutput) {

	def nextReleasePattern = /(?ms).*The next release version is (?<release>.*?)$.*/
	def firstReleasePattern = /(?ms).*There is no previous release, the next release version is (?<release>.*?)$.*/

	def version = ''
	[nextReleasePattern, firstReleasePattern].each { x ->
		if (version == '') {
			def releaseMatch = semanticReleaseOutput =~ x
			if (releaseMatch.matches()) {
				version = releaseMatch.group('release')
			}
		}
	}
	version
}

def runSemanticRelease(token, preview) {

	def dryRun = ''
	if (preview) {
		dryRun = '--dry-run'
	}

	def output = sh(returnStdout: true, script: "export GITHUB_TOKEN=$token; npm install --save-dev conventional-changelog-conventionalcommits@$conventionalChangelogConventionalCommitsVersion; npx semantic-release@$semanticReleaseVersion $dryRun")
	echo output

	getNextSemanticReleaseVersion(output)
}

def getLatestGitHubRelease(token, owner, repo) {

	def latestUrl = "https://api.github.com/repos/$owner/$repo/releases/latest"
	def output = sh(returnStdout: true, script: "curl --silent -H 'Accept: application/vnd.github.v3+json' -H 'Authorization: token $token' $latestUrl")
	echo output

	def tagNamePattern = /(?ms).*"tag_name":\s"(?<release>[^"]+)".*/
	def tagNameMatch = output =~ tagNamePattern

	def version = ''
	if (tagNameMatch.matches()) {
		version = tagNameMatch.group('release')
	}
	version
}

def slackSendMessage(success) {

	def messageColor = 'good'
	def messageStatus = 'succeeded'
	if (!success) {
		messageColor = 'warning'
		messageStatus = 'failed'
	}

	withCredentials([string(credentialsId: 'dockerSlackChannel', variable: 'slackChannel')]) {
		slackSend channel: slackChannel, color: messageColor, message: "Code Dx Docker Compose: Release stage $messageStatus: <${env.BUILD_URL}console|Open>"
	}
}

pipeline {

	options {
		skipDefaultCheckout true // checkout via getRepo()
	}

	agent none

	stages {

		stage('Release') {

			agent {
				label 'powershell-small'
			}

			stages {

				stage('Checkout') {

					steps {

						script {
							currentBuild.displayName = getRepo()
						}
					}
				}

				stage('Test') {

					steps {

						dir ('repo') {

							// note: the -CI parameter sets Run.Exit, but it also creates two files in the working directory
							sh 'pwsh -command "&{ Import-Module Pester; \\$cfg = [PesterConfiguration]::Default; \\$cfg.Run.Exit = \\$true; \\$cfg.Run.Path = \'./.version/test/version.tests.ps1\'; Invoke-Pester -Configuration \\$cfg }"'
						}
					}
				}

				stage('Get Code Dx Version') {

					steps {

						dir ('repo') {

							withCredentials([
								usernamePassword(credentialsId: 'codedx-build-github', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN'),
								string(credentialsId: 'codedxownername', variable: 'GIT_OWNER'),
								string(credentialsId: 'codedxreponame',  variable: 'GIT_REPO')]) {

								script {

									codeDxVersion = getLatestGitHubRelease(GIT_TOKEN, GIT_OWNER, GIT_REPO)
									if (codeDxVersion == '') {
										error('unable to continue because the latest version of Code Dx cannot be found')
									}

									hasCodeDxVersion = sh(returnStdout: true, script: "pwsh -command \"& select-string -path ./docker-compose.yml -Pattern 'image:\\scodedx/codedx-tomcat:$codeDxVersion' -Quiet\"")
									if (hasCodeDxVersion.toBoolean()) {
										error("unable to continue because the repository already references the latest Code Dx version ($codeDxVersion)")
									}
								}
							}
						}
					}
				}

				stage('Confirm') {

					steps {

						milestone ordinal: 1, label: 'Confirm'

						script {

							try {

								timeout(time: 15) {

									// pipeline not triggered by SCM and input response should occur with minimal delay, so invoke input in this stage (leaving container running)
									input message: "Continue with latest Code Dx version ($codeDxVersion)?"
								}
							} catch (err) {

								if (err instanceof org.jenkinsci.plugins.workflow.steps.FlowInterruptedException) {
									error('Timeout occurred while awaiting release confirmation')
								}
								error(err.toString())
							}
						}
					}
				}

				stage('Update Version') {

					steps {

						milestone ordinal: 2, label: 'Confirmed'

						dir ('repo') {

							sh 'git config user.name \'Code Dx Build\' && git config user.email support@codedx.com'
							sh "git checkout ${scm.branches[0]}"
							sh "pwsh ./.version/version.ps1 $codeDxVersion"
							sh 'git add .'
							sh "git commit -m 'feat: update to $codeDxVersion'"

							withCredentials([usernamePassword(credentialsId: 'codedx-build-github', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]){

								// note: pipeline requires 'Suppress automatic SCM triggering' behavior
								sh('''
									git config --local credential.helper "!helper() { echo username=\\$GIT_USERNAME; echo password=\\$GIT_TOKEN; }; helper"
									git push
								''')
							}
						}
					}
				}

				stage('Create Release') {

					steps {

						dir ('repo') {

							withCredentials([usernamePassword(credentialsId: 'codedx-build-github', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {

								script {

									versionReleased = runSemanticRelease(GIT_TOKEN, false)
									if (versionReleased == '') {
										error('Build failed because commits since the last version do not require a new release')
									}
								}
							}
						}
					}
				}
			}

			post {
				success {
					script {
						slackSendMessage(true)
					}
				}
				failure {
					script {
						slackSendMessage(false)
					}
				}
			}
		}
	}
}

