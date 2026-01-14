pipeline {
    agent any

    tools {
        // Jenkins utilisera ces versions installées localement
        // Cela évite le téléchargement internet qui causait ton Timeout
        gradle 'Gradle-8.5'
        jdk 'JDK-21'
    }

    environment {
        SONAR_HOST_URL = 'http://localhost:9000'

        // On récupère le secret "sonar-token" depuis le coffre-fort Jenkins
        SONAR_TOKEN = credentials('sonar-token')

        MAVEN_REPO_USER = 'myMavenRepo'
        // Si ce credential n'existe pas encore, commente la ligne ci-dessous
        MAVEN_REPO_PASSWORD = credentials('f4f4dc35-6f11-4b90-8eeb-1df7ff6677f3')

        SLACK_CHANNEL = '#dev-notifications'
        EMAIL_RECIPIENTS = 'ma_kadoum@esi.dz'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo '✅ Code source récupéré.'
            }
        }

        stage('Test') {
            steps {
                script {
                    echo '--- Lancement des Tests ---'
                    // On utilise 'gradle' (outil Jenkins) et non './gradlew' (wrapper internet)
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        bat 'gradle clean test --no-daemon'
                    }

                    // Publication des résultats de tests
                    junit testResults: '**/build/test-results/test/*.xml', allowEmptyResults: true

                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/tests/test',
                        reportFiles: 'index.html',
                        reportName: 'Test Report'
                    ])
                }
            }
        }

        stage('Code Analysis') {
            steps {
                script {
                    echo '--- Analyse SonarQube ---'
                    withSonarQubeEnv('SonarQube') {
                        // Utilisation de la syntaxe Windows (^) et injection des variables (%)
                        bat """
                            gradle sonar --no-daemon ^
                            -Dsonar.projectKey=tp5 ^
                            -Dsonar.projectName="TP5 Java Project" ^
                            -Dsonar.host.url=%SONAR_HOST_URL% ^
                            -Dsonar.login=%SONAR_TOKEN%
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        // Attend la réponse de SonarQube
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "⛔ Le Quality Gate a échoué: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                echo '--- Création du JAR ---'
                // -x test permet de ne pas relancer les tests déjà faits
                bat 'gradle build -x test --no-daemon'

                archiveArtifacts artifacts: '**/build/libs/*.jar', fingerprint: true, allowEmptyArchive: true
            }
        }

        // Stage optionnel : Déploiement
        /*
        stage('Deploy') {
            steps {
                script {
                    bat """
                        gradle publish --no-daemon ^
                        -PmavenRepoUsername=%MAVEN_REPO_USER% ^
                        -PmavenRepoPassword=%MAVEN_REPO_PASSWORD%
                    """
                }
            }
        }
        */
    }

    post {
        always {
            script {
                echo '🧹 Nettoyage du workspace...'
                cleanWs()
            }
        }
        success {
            script {
                // Tentative d'envoi Slack sécurisée
                try {
                    slackSend(channel: "${env.SLACK_CHANNEL}", color: 'good', message: "✅ Succès: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
                } catch (Exception e) {
                    echo "⚠️ Notification Slack ignorée (Plugin manquant ou erreur config)."
                }
            }
        }
        failure {
            script {
                // Notification Email
                try {
                    emailext (
                        subject: "❌ Échec - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: "Vérifier la console: ${env.BUILD_URL}",
                        to: "${env.EMAIL_RECIPIENTS}"
                    )
                } catch (Exception e) {
                    echo "⚠️ Notification Email ignorée."
                }

                // Tentative d'envoi Slack sécurisée
                try {
                    slackSend(channel: "${env.SLACK_CHANNEL}", color: 'danger', message: "❌ Échec: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
                } catch (Exception e) {
                    echo "⚠️ Notification Slack ignorée."
                }
            }
        }
    }
}