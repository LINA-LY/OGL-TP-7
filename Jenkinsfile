pipeline {
    agent any

    // Variables d'environnement (optionnel mais pratique)
    environment {
        PROJECT_NAME = 'TP7-API'
    }

    stages {

        // ========== PHASE 1: TEST ==========
        stage('Test') {
            steps {
                echo ' Lancement des tests...'

                // Lance les tests unitaires
                bat '.\\gradlew test'

                // Archive les résultats des tests unitaires
                // Jenkins va créer un rapport graphique
                junit 'build/test-results/test/*.xml'

                // Génère le rapport Cucumber
                // ATTENTION: vérifie que tu as bien des fichiers .json dans reports/
                bat '.\\gradlew generateCucumberReports'

                // Publie le rapport Cucumber dans Jenkins
                cucumber buildStatus: 'UNSTABLE',
                         fileIncludePattern: '**/*.json',
                         jsonReportDirectory: 'reports'
            }
        }

        // ========== PHASE 2: CODE ANALYSIS ==========
        stage('Code Analysis') {
            steps {
                echo ' Analyse du code avec SonarQube...'

                // IMPORTANT: SonarQube doit tourner sur localhost:9000
                withSonarQubeEnv('SonarQube') {
                    bat '.\\gradlew sonarqube'
                }
            }
        }

        // ========== PHASE 3: QUALITY GATE ==========
        stage('Code Quality') {
            steps {
                echo ' Vérification des Quality Gates...'

                // Attend le résultat de SonarQube (max 5 minutes)
                // Si échec → le pipeline s'arrête (abortPipeline: true)
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ========== PHASE 4: BUILD ==========
        stage('Build') {
            steps {
                echo '🔨 Construction du projet...'

                // Génère le JAR (skip tests car déjà faits)
                bat '.\\gradlew build -x test'

                // Génère la documentation Javadoc
                bat '.\\gradlew javadoc'

                // Archive les artefacts
                archiveArtifacts artifacts: 'build/libs/*.jar',
                                fingerprint: true
                archiveArtifacts artifacts: 'build/docs/**/*',
                                fingerprint: true

                echo ' JAR et documentation archivés'
            }
        }

        // ========== PHASE 5: DEPLOY ==========
        stage('Deploy') {
            steps {
                echo ' Déploiement sur MyMavenRepo...'

                // ATTENTION: décommenter la section publishing dans build.gradle
                // Et configurer les credentials dans Jenkins
                bat '.\\gradlew publish'

                echo ' Déploiement réussi'
            }
        }
    }

    // ========== PHASE 6: NOTIFICATIONS ==========
    post {
        success {
            echo ' Pipeline terminé avec succès!'

            // Email de succès
            emailext (
                subject: " Déploiement réussi - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2>Déploiement réussi !</h2>
                    <p><strong>Projet:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build:</strong> #${env.BUILD_NUMBER}</p>
                    <p><strong>Statut:</strong> SUCCESS</p>
                    <p><strong>URL:</strong> ${env.BUILD_URL}</p>
                """,
                to: 'ton-email@gmail.com',
                mimeType: 'text/html'
            )

            // Notification Slack (si configuré)
            // slackSend color: 'good',
            //           message: " Déploiement réussi: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }

        failure {
            echo ' Le pipeline a échoué!'

            // Email d'échec
            emailext (
                subject: " Échec du pipeline - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2>Le pipeline a échoué !</h2>
                    <p><strong>Projet:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build:</strong> #${env.BUILD_NUMBER}</p>
                    <p><strong>Statut:</strong> FAILURE</p>
                    <p><strong>Vérifier les logs:</strong> ${env.BUILD_URL}console</p>
                """,
                to: 'ton-email@gmail.com',
                mimeType: 'text/html'
            )
        }

        always {
            echo ' Nettoyage...'
            // Nettoie l'espace de travail si besoin
            // cleanWs()
        }
    }
}