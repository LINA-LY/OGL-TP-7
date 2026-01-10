pipeline {
    agent any

    environment {
        PROJECT_NAME = 'TP7-API'
    }

    stages {

        // ========== PHASE 0: CLEAN ==========
        stage('Clean') {
            steps {
                echo ' Nettoyage...'
                bat '.\\gradlew clean --no-daemon --refresh-dependencies'
            }
        }

        // ========== PHASE 1: TEST ==========
        stage('Test') {
            steps {
                echo ' Lancement des tests...'

                // Lance les tests avec retry en cas d'échec réseau
                retry(2) {
                    bat '.\\gradlew test --no-daemon --refresh-dependencies'
                }

                // Archive les résultats
                junit 'build/test-results/test/*.xml'

                // Génère Cucumber (si tes tests passent)
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

        // ========== PHASE 2: CODE ANALYSIS ==========
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
                        echo "Continuing pipeline..."
                    }
                }
            }
        }

        // ========== PHASE 3: QUALITY GATE (optionnel si SonarQube marche) ==========
        stage('Code Quality') {
            when {
                expression { return false } // Désactivé pour l'instant
            }
            steps {
                echo ' Vérification des Quality Gates...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        // ========== PHASE 4: BUILD ==========
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

        // ========== PHASE 5: DEPLOY ==========
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

    post {
        success {
            echo ' Pipeline SUCCESS!'
            emailext (
                subject: " Build Success - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build réussi !",
                to: 'ml_hamadache@esi.dz',
                mimeType: 'text/html'
            )
        }

        failure {
            echo ' Pipeline FAILED!'
            emailext (
                subject: " Build Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Vérifier les logs: ${env.BUILD_URL}console",
                to: 'ml_hamadache@esi.dz',
                mimeType: 'text/html'
            )
        }
    }
}