/*
Jenkins Pipeline is a suite of plugins which supports implementing and integrating continuous delivery pipelines into Jenkins.
Pipeline provides an extensible set of tools for modeling delivery pipelines "as code" via the Pipeline DSL.
More information can be found on the Jenkins Documentation page https://jenkins.io/doc/
*/

library 'github-utils-shared-library@master'
@Library('github.com/connexta/cx-pipeline-library@master') _
pipeline {
    agent {
        node {
            label 'linux-small'
            customWorkspace "/jenkins/workspace/${env.JOB_NAME}/${env.BUILD_NUMBER}"
        }
    }
    environment {
        DISABLE_DOWNLOAD_PROGRESS_OPTS = '-Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn '
        LINUX_MVN_RANDOM = '-Djava.security.egd=file:/dev/./urandom'
        COVERAGE_EXCLUSIONS = '**/test/**/*,**/itests/**/*,**/*Test*,**/sdk/**/*,**/*.js,**/node_modules/**/*,**/jaxb/**/*,**/wsdl/**/*,**/nces/sws/**/*,**/*.adoc,**/*.txt,**/*.xml'
        GITHUB_USERNAME = 'connexta'
        GITHUB_REPONAME = 'ion-ingest-api'
        GITHUB_KEY = 'ion-ingest-api-github-key'
    }
    parameters {
        booleanParam(name: 'RELEASE', defaultValue: false, description: 'Perform Release?')
        string(name: 'RELEASE_VERSION', defaultValue: 'NA', description: 'The version to release. An NA value will release the current version')
        string(name: 'RELEASE_TAG', defaultValue: 'NA', description: 'The release tag for this version. An NA value will result in <repository-name>-RELEASE_VERSION')
        string(name: 'NEXT_VERSION', defaultValue: 'NA', description: 'The next development version. An NA value will increment the patch version')
    }
    options {
        buildDiscarder(logRotator(numToKeepStr:'25'))
        disableConcurrentBuilds()
        timestamps()
    }
    triggers {
        /*
          Restrict nightly builds to master branch, all others will be built on change only.
          Note: The BRANCH_NAME will only work with a multi-branch job using the github-branch-source
        */
        cron(env.BRANCH_NAME == "master" ? "H H(21-23) * * *" : "")
    }
    stages {
        stage('Setup') {
            steps {
                script {
                    slackSend color: 'good', message: "STARTED: ${JOB_NAME} ${BUILD_NUMBER} ${BUILD_URL}"
                    withCredentials([usernameColonPassword(credentialsId: 'cxbot', variable: 'GITHUB_TOKEN')]) {
                        postCommentIfPR("Internal build has been started. Your results will be available at completion. See build progress in [legacy Jenkins UI](${BUILD_URL}) or in [Blue Ocean UI](${BUILD_URL}display/redirect).", "${GITHUB_USERNAME}", "${GITHUB_REPONAME}", "${GITHUB_TOKEN}")
                        script {
                            //  Clear existing status checks
                            def jsonBlob = getGithubStatusJsonBlob("pending", "${BUILD_URL}display/redirect", "Full Build In Progress...", "CX Jenkins/Full Build")

                            try {
                                //  Check to see if there are multiple parents for the commit. (merged)
                                sh(script: 'if [ `git cat-file -p HEAD | head -n 3 | grep parent | wc -l` -gt 1 ]; then exit 1; else exit 0; fi')
                                //  No error was thrown -> we called exit 0 -> HEAD is not a merge commit/doesn't have multiple parents
                                env.PR_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                            } catch (err) {
                                //  An error was thrown -> we called exit 1 -> HEAD is a merge commit/has multiple parents
                                env.PR_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD~1').trim()
                            }

                            postStatusToHash("${jsonBlob}", "${GITHUB_USERNAME}", "${GITHUB_REPONAME}", "${env.PR_COMMIT}", "${GITHUB_TOKEN}")
                        }
                    }
                    if (params.RELEASE == true) {
                        if (params.RELEASE_VERSION != 'NA') {
                            env.RELEASE_VERSION = params.RELEASE_VERSION
                        } else {
                            echo ("Setting release version to ${getBaseVersion()}")
                            env.RELEASE_VERSION = getBaseVersion()
                        }
                        if (params.RELEASE_TAG != 'NA') {
                            env.RELEASE_TAG = params.RELEASE_TAG
                        } else {
                            echo("Setting release tag")
                            env.RELEASE_TAG = "${env.GITHUB_REPONAME}-${env.RELEASE_VERSION}"
                        }
                        if (params.NEXT_VERSION != 'NA') {
                            env.NEXT_VERSION = params.NEXT_VERSION
                        } else {
                            echo("Setting next version")
                            env.NEXT_VERSION = getDevelopmentVersion()
                        }
                        echo("Release parameters: release-version: ${env.RELEASE_VERSION} release-tag: ${env.RELEASE_TAG} next-version: ${env.NEXT_VERSION}")
                    }
                }
            }
        }
        // The incremental build will be triggered only for PRs. It will build the differences between the PR and the target branch
        stage('Incremental Build') {
            when {
                allOf {
                    expression { env.CHANGE_ID != null }
                    expression { env.CHANGE_TARGET != null }
                }
            }
            parallel {
                stage ('Linux') {
                    steps {
                        withMaven(maven: 'Maven 3.5.4', jdk: 'jdk11', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings', mavenOpts: '${LINUX_MVN_RANDOM}') {
                            sh 'mvn install -B -DskipStatic=true -DskipTests=true $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                            sh 'mvn clean install -B -Dgib.enabled=true -Dgib.referenceBranch=/refs/remotes/origin/$CHANGE_TARGET $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                        }
                    }
                }
            }
        }
        stage('Full Build') {
            when { expression { env.CHANGE_ID == null } }
            parallel {
                stage ('Linux') {
                    steps {
                        withMaven(maven: 'Maven 3.5.4', jdk: 'jdk11', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings', mavenOpts: '${LINUX_MVN_RANDOM}') {
                            script {
                                if (params.RELEASE == true) {
                                    sh "mvn -B -Dtag=${env.RELEASE_TAG} -DreleaseVersion=${env.RELEASE_VERSION} -DdevelopmentVersion=${env.NEXT_VERSION} release:prepare"
                                    env.RELEASE_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                                } else {
                                    sh 'mvn clean install -B $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Quality Analysis') {
            parallel {
                // Sonar stage only runs against master
                stage ('SonarCloud') {
                    when {
                        allOf {
                            expression { env.CHANGE_ID == null }
                            branch 'master'
                            environment name: 'JENKINS_ENV', value: 'prod'
                        }
                    }
                    steps {
                        script {
                            if (params.RELEASE == true) {
                                sh "git checkout ${env.RELEASE_TAG}"
                            }
                        }
                        withMaven(maven: 'M35', jdk: 'jdk11', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings', mavenOpts: '${LINUX_MVN_RANDOM}') {
                            withCredentials([string(credentialsId: 'SonarQubeGithubToken', variable: 'SONARQUBE_GITHUB_TOKEN'), string(credentialsId: 'cxbot-sonarcloud', variable: 'SONAR_TOKEN')]) {
                                script {
                                    sh 'mvn -q -B -Dcheckstyle.skip=true org.jacoco:jacoco-maven-plugin:prepare-agent install sonar:sonar -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=$SONAR_TOKEN  -Dsonar.organization=cx -Dsonar.projectKey=${GITHUB_REPONAME} -Dsonar.exclusions=${COVERAGE_EXCLUSIONS} $DISABLE_DOWNLOAD_PROGRESS_OPTS'

                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Release Tag') {
            when { expression { params.RELEASE == true } }
            steps {
                script {
                    sh "git checkout ${env.RELEASE_COMMIT}"

                    //sshagent doesn't seem to work in multi-branch pipelines so the following hack is needed
                    sh "git remote add ssh-origin git@github.com:connexta/${env.GITHUB_REPONAME}.git"
                    withCredentials([sshUserPrivateKey(credentialsId: "${env.GITHUB_KEY}", keyFileVariable: 'GITHUB_KEY')]) {
                        sh 'echo ssh -i $GITHUB_KEY -l git -o StrictHostKeyChecking=no \\"\\$@\\" > run_ssh.sh'
                        sh 'chmod +x run_ssh.sh'
                        withEnv(["GIT_SSH=${WORKSPACE}/run_ssh.sh"]) {
                            sh "git push ssh-origin HEAD:${env.BRANCH_NAME} && git push ssh-origin ${env.RELEASE_TAG}"
                        }
                    }
                }
            }
        }
        /*
         Deploy stage will only be executed for deployable branches. These include master and any patch branch matching M.m.x format (i.e. 2.10.x, 2.9.x, etc...).
         It will also only deploy in the presence of an environment variable JENKINS_ENV = 'prod'. This can be passed in globally from the jenkins master node settings.
        */
        stage('Deploy') {
            when {
                allOf {
                    expression { env.CHANGE_ID == null }
                    anyOf {
                        expression { params.RELEASE == true }
                        expression { env.BRANCH_NAME ==~ /((?:\d*\.)?\d.x|master)/ }
                    }
                    environment name: 'JENKINS_ENV', value: 'prod'
                }
            }
            steps{
                script {
                    if(params.RELEASE == true) {
                        sh "git checkout ${env.RELEASE_TAG}"
                    }
                }

                withMaven(maven: 'Maven 3.5.4', jdk: 'jdk11', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings', mavenOpts: '${LINUX_MVN_RANDOM}') {
                    sh 'mvn javadoc:aggregate -B -DskipStatic=true -DskipTests=true -nsu $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                    sh 'mvn deploy -B -DskipStatic=true -DskipTests=true -DretryFailedDeploymentCount=10 -nsu $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                }
            }
        }
    }
    post {
        success {
            slackSend color: 'good', message: "SUCCESS: ${JOB_NAME} ${BUILD_NUMBER}"
            withCredentials([usernameColonPassword(credentialsId: 'cxbot', variable: 'GITHUB_TOKEN')]) {
                postCommentIfPR("✅ Build success! See the job results in [legacy Jenkins UI](${BUILD_URL}) or in [Blue Ocean UI](${BUILD_URL}display/redirect).", "${GITHUB_USERNAME}", "${GITHUB_REPONAME}", "${GITHUB_TOKEN}")
                script {
                    def jsonBlob = getGithubStatusJsonBlob("success", "${BUILD_URL}display/redirect", "Full build succeeded!", "CX Jenkins/Full Build")
                    postStatusToHash("${jsonBlob}", "${GITHUB_USERNAME}", "${GITHUB_REPONAME}", "${env.PR_COMMIT}", "${GITHUB_TOKEN}")
                }
            }
        }
        failure {
            slackSend color: '#ea0017', message: "FAILURE: ${JOB_NAME} ${BUILD_NUMBER}. See the results here: ${BUILD_URL}"
            withCredentials([usernameColonPassword(credentialsId: 'cxbot', variable: 'GITHUB_TOKEN')]) {
                postCommentIfPR("❌ Build failure. See the job results in [legacy Jenkins UI](${BUILD_URL}) or in [Blue Ocean UI](${BUILD_URL}display/redirect).", "${GITHUB_USERNAME}", "${GITHUB_REPONAME}", "${GITHUB_TOKEN}")
                script {
                    def jsonBlob = getGithubStatusJsonBlob("failure", "${BUILD_URL}display/redirect", "Full Build Failed!", "CX Jenkins/Full Build")
                    postStatusToHash("${jsonBlob}", "${GITHUB_USERNAME}", "${GITHUB_REPONAME}", "${env.PR_COMMIT}", "${GITHUB_TOKEN}")
                }
            }
        }
        unstable {
            slackSend color: '#ffb600', message: "UNSTABLE: ${JOB_NAME} ${BUILD_NUMBER}. See the results here: ${BUILD_URL}"
            withCredentials([usernameColonPassword(credentialsId: 'cxbot', variable: 'GITHUB_TOKEN')]) {
                postCommentIfPR("⚠️ Build unstable. See the job results in [legacy Jenkins UI](${BUILD_URL}) or in [Blue Ocean UI](${BUILD_URL}display/redirect).", "${GITHUB_USERNAME}", "${GITHUB_REPONAME}", "${GITHUB_TOKEN}")
                script {
                    def jsonBlob = getGithubStatusJsonBlob("failure", "${BUILD_URL}display/redirect", "Full build was unstable!", "CX Jenkins/Full Build")
                    postStatusToHash("${jsonBlob}", "${GITHUB_USERNAME}", "${GITHUB_REPONAME}", "${env.PR_COMMIT}", "${GITHUB_TOKEN}")
                }
            }
        }
    }
}

def getCurrentVersion() {
    def pom = readMavenPom file: 'pom.xml'
    return pom.getVersion()
}

def getBaseVersion() {
    return getCurrentVersion().split('-')[0]
}

def getDevelopmentVersion() {
    def baseVersion = getBaseVersion()
    def patch = baseVersion.substring(baseVersion.lastIndexOf('.')+1).toInteger()+1
    def majMin = baseVersion.substring(0, baseVersion.lastIndexOf('.'))
    def developmentVersion = "${majMin}.${patch}-SNAPSHOT"
    return developmentVersion
}