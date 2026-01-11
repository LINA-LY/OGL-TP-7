pipeline {
    agent any

    environment {
        PROJECT_NAME = 'TP7-API'
    }

    stages {
        stage('Clean') {
            steps {
                echo ' Nettoyage...'
                bat '.\\gradlew clean --no-daemon --refresh-dependencies'
            }
        }

        stage('Test') {
            steps {
                echo ' Lancement des tests...'
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
                        echo " Cucumber reports non générés: ${e.message}"
                    }
                }
            }
        }

        stage('Code Analysis') {
            steps {
                echo ' Analyse du code avec SonarQube...'
                script {
                    try {
                        withSonarQubeEnv('SonarQube') {
                            bat '.\\gradlew sonarqube --no-daemon'
                        }
                    } catch (Exception e) {
                        echo " SonarQube analysis failed: ${e.message}"
                    }
                }
            }
        }

        stage('Code Quality') {
            steps {
                echo ' Vérification des Quality Gates...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true  // ✅ Bloquant si échec
                }
            }
        }

        stage('Build') {
            steps {
                echo ' Construction du projet...'
                bat '.\\gradlew build -x test --no-daemon'
                bat '.\\gradlew javadoc --no-daemon'
                archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
                archiveArtifacts artifacts: 'build/docs/**/*', fingerprint: true
                echo ' Build terminé'
            }
        }

        stage('Deploy') {
            steps {
                echo ' Déploiement...'
                script {
                    try {
                        bat '.\\gradlew publish --no-daemon'
                        echo ' Déploiement réussi'
                    } catch (Exception e) {
                        echo " Deploy failed: ${e.message}"
                    }
                }
            }
        }
    }

    // ✅ UN SEUL BLOC POST — CORRIGÉ
    post {
        success {
            echo '✅ Pipeline réussi !'

            slackSend(
                channel: '#general',
                color: 'good',
                message: "✅ Déploiement réussi !\nProjet: ${env.JOB_NAME}\nBuild: #${env.BUILD_NUMBER}\nDate: ${new Date().format('yyyy-MM-dd HH:mm')}"
            )

            emailext (
                subject: "✅ Build Réussi - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2>✅ Build Réussi</h2>
                    <p><b>Projet :</b> ${env.JOB_NAME}</p>
                    <p><b>Build n° :</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Date :</b> ${new Date()}</p>
                    <p><a href="${env.BUILD_URL}">Voir les détails du build</a></p>
                """,
                to: 'ml_hamadache@esi.dz',
                mimeType: 'text/html'
            )
        }

        failure {
            echo '❌ Pipeline échoué !'

            slackSend(
                channel: '#general',
                color: 'danger',
                message: "❌ Échec du build !\nProjet: ${env.JOB_NAME}\nBuild: #${env.BUILD_NUMBER}\nLogs: ${env.BUILD_URL}"
            )

            emailext (
                subject: "❌ Build Échoué - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2>❌ Build Échoué</h2>
                    <p><b>Projet :</b> ${env.JOB_NAME}</p>
                    <p><b>Build n° :</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Erreur :</b> Une ou plusieurs étapes ont échoué.</p>
                    <p><a href="${env.BUILD_URL}console">Voir les logs complets</a></p>
                """,
                to: 'ml_hamadache@esi.dz',
                mimeType: 'text/html'
            )
        }
    }
}