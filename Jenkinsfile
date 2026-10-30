pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    // A GitHub webhook is the preferred trigger. Polling also catches changes
    // when a webhook is missing or temporarily unavailable.
    triggers {
        pollSCM('H/5 * * * *')
    }

    environment {
        DOCKERHUB_REPOSITORY = 'certifiedunc/spe-calculator'
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
        NOTIFICATION_EMAIL = 'akshat0akt52@gmail.com'
        COMPOSE_FILE = 'docker-compose.yml'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Compose') {
            steps {
                sh 'docker compose config --quiet'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B -ntp clean verify'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: false
                }
            }
        }

        stage('Package') {
            steps {
                archiveArtifacts artifacts: 'target/calculator-0.0.1-SNAPSHOT.jar', fingerprint: true
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    def branch = (env.BRANCH_NAME ?: 'local').replaceAll(/[^A-Za-z0-9_.-]/, '-')
                    env.IMAGE_TAG = "${branch}-${env.BUILD_NUMBER}"
                    env.CALCULATOR_IMAGE = "${env.DOCKERHUB_REPOSITORY}:${env.IMAGE_TAG}"
                }
                sh 'docker build --pull --tag "$CALCULATOR_IMAGE" .'
                sh 'docker compose build proxy'
            }
        }

        stage('Push') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
                                                  usernameVariable: 'DOCKERHUB_USERNAME',
                                                  passwordVariable: 'DOCKERHUB_TOKEN')]) {
                    sh '''
                        set -eu
                        trap 'docker logout >/dev/null 2>&1 || true' EXIT
                        printf '%s' "$DOCKERHUB_TOKEN" | docker login --username "$DOCKERHUB_USERNAME" --password-stdin
                        docker push "$CALCULATOR_IMAGE"
                        docker tag "$CALCULATOR_IMAGE" "$DOCKERHUB_REPOSITORY:latest"
                        docker push "$DOCKERHUB_REPOSITORY:latest"
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    set -eu
                    docker compose pull calculator
                    docker compose up -d --no-build --remove-orphans --wait --wait-timeout 120
                    docker compose ps
                '''
            }
        }
    }

    post {
        success {
            notifyByEmail('SUCCESS')
        }
        failure {
            notifyByEmail('FAILURE')
        }
        unstable {
            notifyByEmail('UNSTABLE')
        }
        aborted {
            notifyByEmail('ABORTED')
        }
        always {
            cleanWs(deleteDirs: true, notFailBuild: true)
        }
    }
}

def notifyByEmail(String status) {
    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
        emailext(
            to: "${env.NOTIFICATION_EMAIL}",
            subject: "${env.JOB_NAME} #${env.BUILD_NUMBER}: ${status}",
            body: "Build: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nStatus: ${status}\nBuild URL: ${env.BUILD_URL}",
            attachLog: status != 'SUCCESS'
        )
    }
}
