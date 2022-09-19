pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '3'))
    }

    environment {
        GPG_KEY_NAME = "${params.gpgKeyName}"
        NEXUS_HOST = "${params.nexusHost}"
        NEXUS_CREDS = credentials('nexus')
        OSSRH_CREDS = credentials('ossrh')
        GRADLE_CMD = './gradlew \
            -Psigning.gnupg.keyName=$GPG_KEY_NAME \
            -PossrhUsername=$OSSRH_CREDS_USR \
            -PossrhPassword=$OSSRH_CREDS_PSW \
            -PnexusHost=$NEXUS_HOST \
            -PnexusUsername=$NEXUS_CREDS_USR \
            -PnexusPassword=$NEXUS_CREDS_PSW'
    }

    stages {
        stage('Checkout') {
            steps {
                this.notifyBuild('STARTED')
                git "${params.repoURL}"
            }
        }

        stage('Build') {
            steps {
                sh "$GRADLE_CMD clean build -x test"
            }
        }

        stage('Test') {
            steps {
                sh "$GRADLE_CMD test"
            }
        }

        stage('Publish') {
            options {
                timeout(time: 4, unit: 'DAYS')
            }
            input {
                message "Should we deploy module to 'Maven Central' and 'Local Nexus'?"
            }
            steps {
                sh "$GRADLE_CMD publish closeAndReleaseStagingRepository"
            }
        }

        stage('Update Warpfleet') {
            steps {
                sh "echo 'wf publish'"
            }
        }
    }
    post {
        success {
            this.notifyBuild('SUCCESSFUL')
        }
        failure {
            this.notifyBuild('FAILURE')
        }
        aborted {
            this.notifyBuild('ABORTED')
        }
        unstable {
            this.notifyBuild('UNSTABLE')
        }
    }
}

void notifyBuild(String buildStatus) {
    // build status of null means successful
    buildStatus = buildStatus ?: 'SUCCESSFUL'
    String subject = "${buildStatus}: Job ${env.JOB_NAME} [${env.BUILD_DISPLAY_NAME}] | ${version}" as String
    String summary = "${subject} (${env.BUILD_URL})" as String
    // Override default values based on build status
    if (buildStatus == 'STARTED') {
        color = 'YELLOW'
        colorCode = '#FFFF00'
    } else if (buildStatus == 'SUCCESSFUL') {
        color = 'GREEN'
        colorCode = '#00FF00'
    } else if (buildStatus == 'PUBLISHED') {
        color = 'BLUE'
        colorCode = '#0000FF'
    } else {
        color = 'RED'
        colorCode = '#FF0000'
    }

    // Send notifications
    this.notifySlack(colorCode, summary, buildStatus)
}

void notifySlack(String color, String message, String buildStatus) {
    // String slackURL = "${params.slackUrl}"
    // String payload = "{\"username\": \"${env.JOB_NAME}\",\"attachments\":[{\"title\": \"${env.JOB_NAME} ${buildStatus}\",\"color\": \"${color}\",\"text\": \"${message}\"}]}" as String
    // sh "curl -X POST -H 'Content-type: application/json' --data '${payload}' ${slackURL}" as String
}