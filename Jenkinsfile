pipeline {
    
    agent any

    tools {
        gradle 'Gradle-8.5'
        jdk 'JDK-11'
    }

    environment {
        SONAR_HOST_URL = 'http://localhost:9000'
        SONAR_TOKEN = credentials('sonar-token')
        GRADLE_USER_HOME = "${WORKSPACE}/.gradle"
        MAVEN_REPO_USER = 'myMavenRepo'
        MAVEN_REPO_PASSWORD = credentials('9Adoum2004')
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
                    echo '========== PHASE TEST =========='

                    // 1. Lancement des tests unitaires
                    echo 'Lancement des tests unitaires...'
                    sh './gradlew clean test --no-daemon'

                    // 2. Archivage des résultats des tests unitaires
                    echo 'Archivage des résultats des tests...'
                    junit '**/build/test-results/test/*.xml'

                    // 3. Génération des rapports Cucumber (si activés)
                    echo 'Génération des rapports de tests...'
                    // Si vous activez Cucumber plus tard:
                    // sh './gradlew cucumberReports --no-daemon'
                }
            }
            post {
                always {
                    // Publication du rapport Jacoco
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/jacoco',
                        reportFiles: 'index.html',
                        reportName: 'JaCoCo Coverage Report'
                    ])

                    // Publication des résultats des tests
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/tests/test',
                        reportFiles: 'index.html',
                        reportName: 'Test Results Report'
                    ])

                    // Si Cucumber est activé:
                    // publishHTML(target: [
                    //     allowMissing: false,
                    //     alwaysLinkToLastBuild: true,
                    //     keepAll: true,
                    //     reportDir: 'build/reports/cucumber',
                    //     reportFiles: 'cucumber-html-reports/overview-features.html',
                    //     reportName: 'Cucumber Report'
                    // ])
                }
            }
        }

        // ===== PHASE CODE ANALYSIS =====
        stage('Code Analysis') {
            steps {
                script {
                    echo '========== PHASE CODE ANALYSIS =========='
                    echo 'Analyse de la qualité du code avec SonarQube...'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ./gradlew sonarqube --no-daemon \
                            -Dsonar.projectKey=tp5 \
                            -Dsonar.projectName="TP5 Java Project" \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        // ===== PHASE CODE QUALITY =====
        stage('Code Quality') {
            steps {
                script {
                    echo '========== PHASE CODE QUALITY =========='
                    echo 'Vérification du Quality Gate...'

                    timeout(time: 5, unit: 'MINUTES') {
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

                    // 1. Génération du fichier JAR
                    echo 'Génération du fichier JAR...'
                    sh './gradlew clean build -x test --no-daemon'

                    // 2. Génération de la documentation
                    echo 'Génération de la documentation Javadoc...'
                    sh './gradlew javadoc --no-daemon'

                    // 3. Archivage du JAR et de la documentation
                    echo 'Archivage des artefacts...'
                    archiveArtifacts artifacts: '**/build/libs/*.jar',
                                   fingerprint: true,
                                   allowEmptyArchive: false

                    archiveArtifacts artifacts: '**/build/docs/javadoc/**',
                                   fingerprint: true,
                                   allowEmptyArchive: false

                    publishHTML(target: [
                        allowMissing: false,
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
                    echo 'Déploiement vers mymavenrepo.com...'

                    // Configuration des credentials pour Gradle
                    sh """
                        ./gradlew publish --no-daemon \
                        -PmavenRepoUsername=${MAVEN_REPO_USER} \
                        -PmavenRepoPassword=${MAVEN_REPO_PASSWORD}
                    """

                    echo "✓ Déploiement réussi sur mymavenrepo.com (Build #${BUILD_NUMBER})"
                }
            }
        }

        // ===== PHASE NOTIFICATION (SUCCÈS) =====
        stage('Notification') {
            steps {
                script {
                    echo '========== PHASE NOTIFICATION =========='

                    def buildUrl = "${env.BUILD_URL}"
                    def projectName = "${env.JOB_NAME}"
                    def buildNumber = "${env.BUILD_NUMBER}"

                    // Notification Email
                    emailext(
                        subject: "✓ Déploiement réussi - ${projectName} #${buildNumber}",
                        body: """
                            <h2 style="color: green;">✓ Déploiement réussi!</h2>
                            <p><strong>Projet:</strong> ${projectName}</p>
                            <p><strong>Build:</strong> #${buildNumber}</p>
                            <p><strong>Statut:</strong>SUCCESS</p>
                            <p><strong>Date:</strong> ${new Date()}</p>
                            <hr>
                            <h3>Détails du déploiement:</h3>
                            <ul>
                                <li>✓ Tests unitaires: Tous passés avec succès</li>
                                <li>✓ Couverture de code: Rapport JaCoCo généré</li>
                                <li>✓ Quality Gate SonarQube: Validé</li>
                                <li>✓ Artefact JAR: Déployé sur mymavenrepo.com</li>
                                <li>✓ Documentation Javadoc: Générée et archivée</li>
                            </ul>
                            <p><a href="${buildUrl}">Voir les détails du build</a></p>
                            <p><a href="${buildUrl}JaCoCo_Coverage_Report/">Voir le rapport de couverture</a></p>
                            <p><a href="${buildUrl}Javadoc_Documentation/">Voir la documentation Javadoc</a></p>
                        """,
                        to: "${EMAIL_RECIPIENTS}",
                        mimeType: 'text/html'
                    )

                    // Notification Slack
                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: 'good',
                        message: """
                            :white_check_mark: *Déploiement réussi!*
                            *Projet:* ${projectName}
                            *Build:* #${buildNumber}
                            *Statut:* SUCCESS

                            *Détails:*
                            • Tests unitaires passés
                            • Quality Gate validé
                            • JAR déployé sur mymavenrepo.com

                            *Liens:*
                            • <${buildUrl}|Détails du build>
                            • <${buildUrl}JaCoCo_Coverage_Report/|Couverture de code>
                            • <${buildUrl}Javadoc_Documentation/|Documentation>
                        """
                    )

                    echo '✓ Notifications envoyées avec succès'
                }
            }
        }
    }

    // ===== GESTION DES ÉCHECS =====
    post {
        failure {
            script {
                echo '========== ÉCHEC DU PIPELINE =========='

                def buildUrl = "${env.BUILD_URL}"
                def projectName = "${env.JOB_NAME}"
                def buildNumber = "${env.BUILD_NUMBER}"
                def failedStage = "${env.STAGE_NAME}"

                // Notification Email en cas d'échec
                emailext(
                    subject: "✗ Échec du pipeline - ${projectName} #${buildNumber}",
                    body: """
                        <h2 style="color: red;">✗ Échec du pipeline!</h2>
                        <p><strong>Projet:</strong> ${projectName}</p>
                        <p><strong>Build:</strong> #${buildNumber}</p>
                        <p><strong>Statut:</strong> FAILURE</p>
                        <p><strong>Phase échouée:</strong> ${failedStage}</p>
                        <p><strong>Date:</strong> ${new Date()}</p>
                        <hr>
                        <h3>Action requise:</h3>
                        <p style="color: red; font-weight: bold;">L'équipe de développement doit intervenir immédiatement!</p>
                        <p>Veuillez consulter les logs pour identifier la cause de l'échec.</p>

                        <h4>Causes possibles selon la phase:</h4>
                        <ul>
                            <li><strong>Test:</strong> Tests unitaires échoués ou problème de génération de rapports</li>
                            <li><strong>Code Analysis:</strong> Problème de connexion à SonarQube</li>
                            <li><strong>Code Quality:</strong> Quality Gate non validé (dette technique trop élevée)</li>
                            <li><strong>Build:</strong> Erreur de compilation ou génération du JAR</li>
                            <li><strong>Deploy:</strong> Problème de connexion à mymavenrepo.com ou credentials invalides</li>
                        </ul>

                        <p><a href="${buildUrl}console">Voir les logs complets du build</a></p>
                    """,
                    to: "${EMAIL_RECIPIENTS}",
                    mimeType: 'text/html'
                )

                // Notification Slack en cas d'échec
                slackSend(
                    channel: "${SLACK_CHANNEL}",
                    color: 'danger',
                    message: """
                        :x: *Échec du pipeline!*
                        *Projet:* ${projectName}
                        *Build:* #${buildNumber}
                        *Phase échouée:* ${failedStage}
                        *Statut:* FAILURE

                        :warning: *Action requise immédiatement!*

                        <${buildUrl}console|Voir les logs>
                    """
                )
            }
        }

        unstable {
            script {
                echo '========== BUILD INSTABLE =========='

                slackSend(
                    channel: "${SLACK_CHANNEL}",
                    color: 'warning',
                    message: """
                        :warning: *Build instable*
                        *Projet:* ${env.JOB_NAME}
                        *Build:* #${env.BUILD_NUMBER}

                        Certains tests ont échoué mais le build a continué.
                        <${env.BUILD_URL}|Voir les détails>
                    """
                )
            }
        }

        success {
            echo '========== PIPELINE TERMINÉ AVEC SUCCÈS =========='
        }

        always {
            echo 'Nettoyage des ressources...'
            // Publication des rapports JaCoCo pour l'analyse de tendances
            jacoco(
                execPattern: '**/build/jacoco/*.exec',
                classPattern: '**/build/classes/java/main',
                sourcePattern: '**/src/main/java'
            )

            // Nettoyage de l'espace de travail
            cleanWs(
                deleteDirs: true,
                patterns: [
                    [pattern: '.gradle', type: 'INCLUDE'],
                    [pattern: 'build', type: 'INCLUDE']
                ]
            )
        }
    }
}