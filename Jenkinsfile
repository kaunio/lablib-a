pipeline {
    agent { label 'maven' }

    tools {
        maven 'maven-3'
        jdk   'java-17'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '3', artifactNumToKeepStr: '3'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds(abortPrevious: true)
        skipDefaultCheckout(false)
    }

    triggers {
       // snapshotDependencies()
    }

    environment {
        MAVEN_OPTS   = '-Xmx1g -Djansi.force=true'
        MVN          = 'mvn -B -ntp -e'
        IS_MAIN      = "${env.BRANCH_NAME == 'main'}"
        IS_RELEASE   = "${env.TAG_NAME != null}"
    }

    stages {

        stage('Version') {
            steps {
                script {
                    if (env.TAG_NAME) {
                        // tag v3.4.0 -> 3.4.0
                        env.REVISION = env.TAG_NAME.replaceFirst(/^v/, '')
                    } else if (env.BRANCH_NAME == 'main') {
                        env.REVISION = sh(returnStdout: true,
                                          script: './ci/next-version.sh --snapshot').trim()
                    } else {
                        def safe = env.BRANCH_NAME.replaceAll(/[^A-Za-z0-9._-]/, '-')
                        env.REVISION = "0.0.0-${safe}-SNAPSHOT"
                    }
                    currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.REVISION}"
                }
            }
        }

        stage('Build') {
            steps {
                withMaven(maven: 'maven-3',
                          jdk: 'java-17',
                          mavenSettingsConfig: 'nexus-settings',
                          mavenLocalRepo: '.m2repo',
                          publisherStrategy: 'EXPLICIT',
                          options: [
                              artifactsPublisher(disabled: false),
                              junitPublisher(healthScaleFactor: 1.0),
                              pipelineGraphPublisher(lifecycleThreshold: 'deploy'),
                              dependenciesFingerprintPublisher(includeSnapshotVersions: true),
                              jgivenPublisher(disabled: true),
                              concordionPublisher(disabled: true)
                          ]) {

                    sh "${MVN} -Drevision=${env.REVISION} clean verify"
                }
            }
        }
/*
        stage('Quality') {
            when { anyOf { branch 'main'; changeRequest() } }
            steps {
                withSonarQubeEnv('sonar') {
                    sh "${MVN} -Drevision=${env.REVISION} sonar:sonar"
                }
            }
        }

        stage('Deploy SNAPSHOT') {
            when {
                allOf {
                    branch 'main'
                    not { buildingTag() }
                }
            }
            steps {
                withMaven(maven: 'Maven 3.9',
                          jdk: 'JDK 21',
                          mavenSettingsConfig: 'nexus-settings',
                          mavenLocalRepo: '.m2repo',
                          publisherStrategy: 'EXPLICIT',
                          options: [ pipelineGraphPublisher(lifecycleThreshold: 'deploy') ]) {

                    sh "${MVN} -Drevision=${env.REVISION} -DskipTests deploy"
                }
            }
        }

        stage('Release') {
            when { buildingTag() }
            steps {
                withMaven(maven: 'Maven 3.9',
                          jdk: 'JDK 21',
                          mavenSettingsConfig: 'nexus-settings',
                          mavenLocalRepo: '.m2repo',
                          publisherStrategy: 'EXPLICIT',
                          options: [ pipelineGraphPublisher(lifecycleThreshold: 'deploy') ]) {

                    sh "${MVN} -Drevision=${env.REVISION} -DskipTests deploy"
                }
            }
            post {
                success {
                    // nudge Renovate so the BOM picks this up without waiting for the poll
                    withCredentials([string(credentialsId: 'renovate-token', variable: 'TOK')]) {
                        sh '''curl -sf -X POST "$RENOVATE_TRIGGER_URL" \
                                -H "Authorization: Bearer $TOK" \
                                -d '{"repository":"acme/platform"}' '''
                    }
                }
            }
        }
    */
    }

    post {
        always {
            recordIssues tools: [spotBugs(), errorProne()], qualityGates: [[threshold: 1, type: 'NEW', criticality: 'NOTE']]
            cleanWs(deleteDirs: true, notFailBuild: true, patterns: [[pattern: '.m2repo/**', type: 'EXCLUDE']])
        }
        failure {
            slackSend channel: '#builds', color: 'danger',
                      message: "${env.JOB_NAME} #${env.BUILD_NUMBER} failed: ${env.BUILD_URL}"
        }
        fixed {
            slackSend channel: '#builds', color: 'good',
                      message: "${env.JOB_NAME} back to green"
        }
    }
}
