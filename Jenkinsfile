#!/usr/bin/env groovy
import hudson.model.*

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
        GRADLE_CMD = './gradlew -Psigning.gnupg.keyName=$GPG_KEY_NAME -PossrhUsername=$OSSRH_CREDS_USR -PossrhPassword=$OSSRH_CREDS_PSW -PnexusHost=$NEXUS_HOST  -PnexusUsername=$NEXUS_CREDS_USR -PnexusPassword=$NEXUS_CREDS_PSW'
        version = ""
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    version = ""
                }
                this.notifyBuild('STARTED')
                git poll: false, url: "${params.repoURL}"
                script {
                    version = getGitVersion()
                }
            }
        }

        stage('Build') {
            steps {
                sh "$GRADLE_CMD clean build"
            }
        }

        stage('Test') {
            steps {
                sh "$GRADLE_CMD test"
            }
        }

        stage('Package') {
            steps {
                sh "$GRADLE_CMD jar shadowJar sourcesJar javadocJar"
                archiveArtifacts "build/libs/*.jar"
            }
        }

        stage('Deploy to SenX\' Nexus') {
            options {
                timeout(time: 2, unit: 'HOURS')
            }
            input {
                message "Should we deploy libs?"
            }
            steps {
                sh "$GRADLE_CMD publishJarPublicationToNexusRepository" + (params.publishUberJar ? ' publishUberJarPublicationToNexusRepository' : '')
            }
        }

        stage('Deploy to Maven Central') {
            when {
                expression { return isItATagCommit() }
                beforeInput true
            }
            options {
                timeout(time: 2, unit: 'HOURS')
            }
            input {
                message 'Should we deploy to Maven Central?'
            }
            steps {
                sh "$GRADLE_CMD publishJarPublicationToMavenRepository" + (params.publishUberJar ? ' publishUberJarPublicationToMavenRepository' : '')
                sh "$GRADLE_CMD closeRepository"
                sh "$GRADLE_CMD releaseRepository"
                this.notifyBuild('PUBLISHED')
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
    String slackURL = "${params.slackUrl}"
    String payload = "{\"username\": \"${env.JOB_NAME}\",\"attachments\":[{\"title\": \"${env.JOB_NAME} ${buildStatus}\",\"color\": \"${color}\",\"text\": \"${message}\"}]}" as String
    sh "curl -X POST -H 'Content-type: application/json' --data '${payload}' ${slackURL}" as String
}

String getGitVersion() {
    return sh(returnStdout: true, script: 'git describe --abbrev=0 --tags').trim()
}

boolean isItATagCommit() {
    String lastCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
    String tag = sh(returnStdout: true, script: "git show-ref --tags -d | grep ^${lastCommit} | sed -e 's,.* refs/tags/,,' -e 's/\\^{}//'").trim()
    return tag != ''
}