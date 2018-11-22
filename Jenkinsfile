/*
* This program and the accompanying materials are made available under the terms of the
* Eclipse Public License v2.0 which accompanies this distribution, and is available at
* https://www.eclipse.org/legal/epl-v20.html
*
* SPDX-License-Identifier: EPL-2.0
*
* Copyright Contributors to the Zowe Project.
*
*/

/**
 * List of people who will get all emails for master builds
 */
def MASTER_RECIPIENTS_LIST = "cc:Mark.Ackert@broadcom.com"

/**
 * The user's name for git commits
 */
def GIT_USER_NAME = 'zowe-robot'

/**
 * The user's email address for git commits
 */
def GIT_USER_EMAIL = 'zowe.robot@gmail.com'

/**
 * The base repository url for github
 */
def GIT_REPO_URL = 'github.com/zowe/zowe-cli-standalone-package.git'

/**
 * The credentials id field for the authorization token for GitHub stored in Jenkins
 */
def GIT_CREDENTIALS_ID = 'zowe-robot-github'

/**
 * A command to be run that gets the current revision pulled down
 */
def GIT_REVISION_LOOKUP = 'git log -n 1 --pretty=format:%h'

/**
 * The credentials id field for the artifactory username and password
 */
def ARTIFACTORY_CREDENTIALS_ID = 'GizaArtifactory'

/**
 * The email address for the artifactory
 */
def ARTIFACTORY_EMAIL = GIT_USER_EMAIL

/**
* The Artifactory API URL which contains builds of the Zowe CLI
*/
def GIZA_ARTIFACTORY_URL = "https://gizaartifactory.jfrog.io/gizaartifactory/api/npm/npm-local-release/"

pipeline {
    agent {
        label 'ca-jenkins-agent'
    }

    stages {
        /************************************************************************
         * STAGE
         * -----
         * Build Zowe CLI Bundle
         *
         * TIMEOUT
         * -------
         * 10 Minutes
         *
         * EXECUTION CONDITIONS
         * --------------------
         * - Always
         *
         * DECRIPTION
         * ----------
         * Gets the latest version of the Zowe CLI and Zowe CLI Plugins from Giza
         * Artifactory. Creates an archive with 'fat' versions of the CLI and Plugins -
         *  dependencies are bundled.
         *
         * OUTPUTS
         * -------
         * A Zowe CLI Archive containing Zowe CLI, Zowe CLI DB2 Plugin, Zowe CLI CICS Plugin.
         ************************************************************************/
        stage('Create Zowe CLI Bundle') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
            
                    sh "npm set registry https://registry.npmjs.org/"
                    sh "npm set @brightside:registry ${GIZA_ARTIFACTORY_URL}"
                    withCredentials([usernamePassword(credentialsId: ARTIFACTORY_CREDENTIALS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh "./scripts/npm_login.sh $USERNAME $PASSWORD \"$ARTIFACTORY_EMAIL\" '--registry=${GIZA_ARTIFACTORY_URL} --scope=@brightside'"
                    }
                    sh "npm install jsonfile"

                    script {
                        sh "npm pack @brightside/db2@beta"
                        sh "npm pack @brightside/core@beta"
                        sh "npm pack @brightside/cics@beta"
                        sh "./scripts/repackage_bundle.sh *.tgz"
                    }

                    archiveArtifacts artifacts: 'zowe-cli-bundle.zip'
                }
            }
        }
    }
    post {
        /************************************************************************
         * POST BUILD ACTION
         *
         * Sends out emails for the deployment status
         *
         * If the build fails, the build number and a link to the build will be present for further investigation.
         *
         * The build workspace is deleted after the build completes.
         ************************************************************************/
        always {
            script {
                sh 'npm logout || exit 0'

                def buildStatus = currentBuild.currentResult
                try {
                    def previousBuild = currentBuild.getPreviousBuild()
                    def recipients = "${MASTER_RECIPIENTS_LIST}"

                    def subject = "${currentBuild.currentResult}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
                    def consoleOutput = """
                    <p>Branch: <b>${BRANCH_NAME}</b></p>
                    <p>Check console output at "<a href="${RUN_DISPLAY_URL}">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>"</p>
                    """

                    def details = ""

                    if (previousBuild == null) {
                        details = "<p>Initial build for new branch.</p>"
                    } else if (currentBuild.resultIsBetterOrEqualTo(BUILD_SUCCESS) && previousBuild.resultIsWorseOrEqualTo(BUILD_UNSTABLE)) {
                        details = "<p>Build returned to normal.</p>"
                    }

                    // Issue #53 - Previously if the first build for a branch failed, logs would not be captured.
                    //             Now they do!
                    if (currentBuild.resultIsWorseOrEqualTo(BUILD_UNSTABLE)) {
                        // Archives any test artifacts for logging and debugging purposes
                        archiveArtifacts allowEmptyArchive: true, artifacts: '__tests__/__results__/**/*.log'
                        details = "${details}<p>Build Failure.</p>"
                    }

                    if (BRANCH_NAME == MASTER_BRANCH) {
                        recipients = MASTER_RECIPIENTS_LIST

                        details = "${details}<p>A build of master has finished.</p>"

                        if (GIT_SOURCE_UPDATED == "true") {
                            details = "${details}<p>The pipeline was able to automatically bump the pre-release version in git</p>"
                        } else {
                            // Most likely another PR was merged to master before we could do the commit thus we can't
                            // have the pipeline automatically do it
                            details = """${details}<p>The pipeline was unable to automatically bump the pre-release version in git.
                            <b>THIS IS LIKELY NOT AN ISSUE WITH THE BUILD</b> as all the tests have to pass to get to this point.<br/><br/>
                            <b>Possible causes of this error:</b>
                            <ul>
                                <li>A commit was made to <b>${MASTER_BRANCH}</b> during the current run.</li>
                                <li>The user account tied to the build is no longer valid.</li>
                                <li>The remote server is experiencing issues.</li>
                            </ul>
                            <i>THIS BUILD WILL BE MARKED AS A FAILURE AS WE CANNOT GUARENTEE THAT THE PROBLEM DOES NOT LIE IN THE
                            BUILD AND CORRECTIVE ACTION MAY NEED TO TAKE PLACE.</i>
                            </p>"""
                        }
                    }

                    if (details != "") {
                        echo "Sending out email with details"
                        emailext(
                                subject: subject,
                                to: recipients,
                                body: "${details} ${consoleOutput}",
                                recipientProviders: [[$class: 'DevelopersRecipientProvider'],
                                                        [$class: 'UpstreamComitterRecipientProvider'],
                                                        [$class: 'CulpritsRecipientProvider'],
                                                        [$class: 'RequesterRecipientProvider']]
                        )
                    }
                } catch (e) {
                    echo "Experienced an error sending an email for a ${buildStatus} build"
                    currentBuild.result = buildStatus
                }
            }
        }
    }
}