pipeline {
    agent any

    environment {
        PROJECT_NAME = 'TP7-API'
        // Secrets injectés depuis Jenkins Credentials
        SLACK_WEBHOOK_URL = credentials('slack-webhook')
        EMAIL_FROM        = credentials('email-from')
        EMAIL_PASSWORD    = credentials('email-password')
        EMAIL_TO          = credentials('email-to')
    }

    stages {
        stage('Clean') {
            steps {
                bat '.\\gradlew clean --no-daemon --refresh-dependencies'
            }
        }

        stage('Test') {
            steps {
                retry(2) {
                    bat '.\\gradlew test --no-daemon --refresh-dependencies'
                }
                junit 'build/test-results/test/*.xml'

                script {
                    try {
                        bat '.\\gradlew generateCucumberReports --no-daemon'
                        cucumber buildStatus: 'UNSTABLE',
                                 fileIncludePattern: '**/*.json',
                                 jsonReportDirectory: 'reports'
                    } catch (Exception e) {
                        echo "Cucumber reports non générés: ${e.message}"
                    }
                }
            }
        }

        stage('Code Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        bat '.\\gradlew sonarqube --no-daemon'
                    }
                }
            }
        }

        stage('Code Quality') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                bat '.\\gradlew build -x test --no-daemon'
                bat '.\\gradlew javadoc --no-daemon'
                archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
                archiveArtifacts artifacts: 'build/docs/**/*', fingerprint: true
            }
        }

        // ========== PHASE DÉPLOIEMENT + NOTIFICATIONS ==========
        stage('Deploy and Notify') {
            steps {
                // Lance 'publish' → déclenche automatiquement notifyEmail et notifySlack
                bat '.\\gradlew publish ' +
                    '-PslackWebhook=%SLACK_WEBHOOK_URL% ' +
                    '-PemailFrom=%EMAIL_FROM% ' +
                    '-PemailPassword=%EMAIL_PASSWORD% ' +
                    '-PemailTo=%EMAIL_TO% ' +
                    '--no-daemon'
            }
        }
    }

    post {
        success {
            echo ' Pipeline terminé avec succès'
        }
        failure {
            echo ' Pipeline échoué'
        }
    }
}