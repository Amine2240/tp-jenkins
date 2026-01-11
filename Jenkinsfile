pipeline {
    agent any

    tools {
        // Assure-toi que ces noms correspondent exactement à "Manage Jenkins > Global Tool Configuration"
        gradle 'Gradle-8.5'
        jdk 'JDK-21'
    }

    environment {
        // URL de ton serveur Sonar
        SONAR_HOST_URL = 'http://localhost:9000'

        // Les credentials sont injectés en tant que variables d'environnement
        // Gradle détectera automatiquement 'SONAR_TOKEN' s'il est présent dans l'env
        SONAR_TOKEN = "sonar-token"

        GRADLE_USER_HOME = "${WORKSPACE}/.gradle"
        MAVEN_REPO_USER = 'myMavenRepo'
        MAVEN_REPO_PASSWORD = credentials('f4f4dc35-6f11-4b90-8eeb-1df7ff6677f3')

        SLACK_CHANNEL = '#dev-notifications'
        EMAIL_RECIPIENTS = 'ma_kadoum@esi.dz'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo 'Code récupéré avec succès'
            }
        }

        // ===== PHASE TEST =====
        stage('Test') {
            steps {
                script {
                    echo '========== Phase Test =========='

                    // 1. Tests unitaires
                    // Astuce : --info aide à déboguer si les tests plantent
                    bat './gradlew clean test --no-daemon'

                    // 2. Archivage JUnit
                    junit '**/build/test-results/test/*.xml'

                    // Rapport HTML des tests
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/tests/test',
                        reportFiles: 'index.html',
                        reportName: 'Test Report'
                    ])

                    // 3. Tests Cucumber
                    // Utilisation de try/catch ou continue pour ne pas bloquer si les tests cucumber échouent
                    // Ici on lance simplement la tâche, Gradle gérera le statut
                    bat './gradlew test --no-daemon'

                    // Note: Si tu as une tâche spécifique pour cucumber (ex: ./gradlew cucumber), utilise-la ici.
                    // Sinon, 'test' lance souvent tout.

                    cucumber buildStatus: 'UNSTABLE',
                            reportTitle: 'Rapport Cucumber',
                            fileIncludePattern: 'reports/*.json',
                            trendsLimit: 10

                    publishHTML([
                        allowMissing: true, // true car Cucumber ne génère pas toujours le HTML si échec
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/cucumber/generated/html',
                        reportFiles: 'overview-features.html',
                        reportName: 'Cucumber HTML Report'
                    ])
                }
            }
            post {
                always {
                    // Publication du rapport JaCoCo (HTML uniquement)
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/jacoco/test/html', // Chemin standard Gradle 8+
                        reportFiles: 'index.html',
                        reportName: 'JaCoCo Coverage Report'
                    ])
                }
            }
        }

        // ===== PHASE CODE ANALYSIS =====
        stage('Code Analysis') {
            steps {
                script {
                    echo '========== PHASE CODE ANALYSIS =========='

                    // IMPORTANT : Le nom 'SonarQube' doit exister dans :
                    // Manage Jenkins > Configure System > SonarQube servers
                    withSonarQubeEnv('SonarQube') {
                        // CORRECTION SÉCURITÉ :
                        // On n'interpole pas ${SONAR_TOKEN} ici.
                        // Gradle 8.5+ détecte automatiquement la variable d'env "SONAR_TOKEN"
                        // définie dans le bloc environment.
                        // On remplace 'sonarqube' par 'sonar'.
                        bat """
                            ./gradlew sonar --no-daemon ^
                            -Dsonar.projectKey=tp5 ^
                            -Dsonar.projectName="TP5 Java Project" ^
                            -Dsonar.host.url=%SONAR_HOST_URL%
                            -Dsonar.login=%SONAR_TOKEN%
                        """
                        // Note : ^ est le caractère de saut de ligne pour Windows cmd
                    }
                }
            }
        }

        // ===== PHASE CODE QUALITY =====
        stage('Code Quality') {
            steps {
                script {
                    echo '========== PHASE CODE QUALITY =========='
                    timeout(time: 5, unit: 'MINUTES') {
                        // Attend que SonarQube renvoie le webhook à Jenkins
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Le Quality Gate a échoué avec le statut: ${qg.status}"
                        } else {
                            echo "✓ Quality Gate validé avec succès!"
                        }
                    }
                }
            }
        }

        // ===== PHASE BUILD =====
        stage('Build') {
            steps {
                script {
                    echo '========== PHASE BUILD =========='
                    // -x test pour éviter de relancer les tests déjà faits
                    bat './gradlew clean build -x test --no-daemon'

                    echo 'Génération Javadoc...'
                    bat './gradlew javadoc --no-daemon'

                    // Archivage
                    archiveArtifacts artifacts: '**/build/libs/*.jar', fingerprint: true
                    archiveArtifacts artifacts: '**/build/docs/javadoc/**', fingerprint: true, allowEmptyArchive: true

                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/docs/javadoc',
                        reportFiles: 'index.html',
                        reportName: 'Javadoc Documentation'
                    ])
                }
            }
        }

        // ===== PHASE DEPLOY =====
        stage('Deploy') {
            steps {
                script {
                    echo '========== PHASE DEPLOY =========='

                    // CORRECTION SÉCURITÉ :
                    // On utilise %VAR% (syntaxe Windows) pour que le mot de passe
                    // ne soit pas résolu par Groovy (visible) mais par le shell (masqué)
                    bat """
                        ./gradlew publish --no-daemon ^
                        -PmavenRepoUsername=%MAVEN_REPO_USER% ^
                        -PmavenRepoPassword=%MAVEN_REPO_PASSWORD%
                    """

                    echo "✓ Déploiement réussi (Build #${env.BUILD_NUMBER})"
                }
            }
        }

        // ===== PHASE NOTIFICATION (SUCCÈS) =====
        stage('Notification') {
            steps {
                script {
                    // Utilisation correcte de env.VARIABLE
                    def buildUrl = "${env.BUILD_URL}"
                    def projectName = "${env.JOB_NAME}"
                    def buildNumber = "${env.BUILD_NUMBER}"

                    emailext(
                        subject: "✓ Déploiement réussi - ${projectName} #${buildNumber}",
                        body: """
                            <h2 style="color: green;">✓ Déploiement réussi!</h2>
                            <p>Projet: ${projectName}</p>
                            <p>Build: #${buildNumber}</p>
                            <p><a href="${buildUrl}">Voir le build</a></p>
                        """,
                        // CORRECTION : env.EMAIL_RECIPIENTS
                        to: "${env.EMAIL_RECIPIENTS}",
                        mimeType: 'text/html'
                    )

                    slackSend(
                        channel: "${env.SLACK_CHANNEL}",
                        color: 'good',
                        message: ":white_check_mark: *Succès:* ${projectName} #${buildNumber}\n(<${buildUrl}|Ouvrir>)"
                    )
                }
            }
        }
    }

    // ===== GESTION DES ÉCHECS =====
    post {
        failure {
            script {
                // CORRECTION : env.VARIABLE partout
                def buildUrl = "${env.BUILD_URL}"
                def projectName = "${env.JOB_NAME}"
                def buildNumber = "${env.BUILD_NUMBER}"
                def failedStage = "${env.STAGE_NAME}" // Note: STAGE_NAME n'est pas toujours dispo en post global, mais on tente

                emailext(
                    subject: "✗ Échec - ${projectName} #${buildNumber}",
                    body: """
                        <h2 style="color: red;">✗ Échec du pipeline</h2>
                        <p>Phase: ${failedStage}</p>
                        <p><a href="${buildUrl}console">Voir les logs</a></p>
                    """,
                    // CORRECTION : env.EMAIL_RECIPIENTS
                    to: "${env.EMAIL_RECIPIENTS}",
                    mimeType: 'text/html'
                )

                slackSend(
                    channel: "${env.SLACK_CHANNEL}",
                    color: 'danger',
                    message: ":x: *Échec:* ${projectName} #${buildNumber}\nCheck logs: <${buildUrl}console|Console>"
                )
            }
        }

        always {
            echo 'Nettoyage...'
            // cleanWs simple est plus robuste sous Windows que deleteDirs avec patterns complexes
            cleanWs()
        }
    }
}