pipeline {
    agent any

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

/*
    triggers {
       snapshotDependencies()
    }
*/
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
                        env.REVISION = "0.0.0-blarg-SNAPSHOT"
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
                              dependenciesFingerprintPublisher(includeSnapshotVersions: true)
                          ]) {

                    sh "${MVN} -Drevision=${env.REVISION} clean verify"
                }
            }
        }
    }

    post {
        always {
            cleanWs(deleteDirs: true, notFailBuild: true, patterns: [[pattern: '.m2repo/**', type: 'EXCLUDE']])
        }
    }
}
