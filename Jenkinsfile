pipeline {
    agent any

    tools {
        gradle 'Gradle-8.5'
        jdk 'JDK-21'
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
                    echo 'Lancement des tests unitaires...'
                    sh './gradlew clean test --no-daemon'

                    echo 'Archivage des résultats des tests...'
                    junit '**/build/test-results/test/*.xml'
                }
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/tests/test',
                        reportFiles: 'index.html',
                        reportName: 'Test Results Report'
                    ])
                }
            }
        }

        // ===== PHASE CODE ANALYSIS =====
        stage('Code Analysis') {
            steps {
                script {
                    echo '========== PHASE CODE ANALYSIS =========='
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
                    timeout(time: 5, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Le Quality Gate a échoué : ${qg.status}"
                        }
                        echo '✓ Quality Gate validé'
                    }
                }
            }
        }

        // ===== PHASE BUILD =====
        stage('Build') {
            steps {
                script {
                    echo '========== PHASE BUILD =========='
                    sh './gradlew clean build -x test --no-daemon'
                    sh './gradlew javadoc --no-daemon'

                    archiveArtifacts artifacts: '**/build/libs/*.jar', fingerprint: true
                    archiveArtifacts artifacts: '**/build/docs/javadoc/**', fingerprint: true

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
                    sh """
                        ./gradlew publish --no-daemon \
                        -PmavenRepoUsername=${MAVEN_REPO_USER} \
                        -PmavenRepoPassword=${MAVEN_REPO_PASSWORD}
                    """
                }
            }
        }

        // ===== PHASE NOTIFICATION =====
        stage('Notification') {
            steps {
                script {
                    def buildUrl = env.BUILD_URL
                    def projectName = env.JOB_NAME
                    def buildNumber = env.BUILD_NUMBER

                    emailext(
                        subject: "✓ Déploiement réussi - ${projectName} #${buildNumber}",
                        mimeType: 'text/html',
                        to: EMAIL_RECIPIENTS,
                        body: """
                            <h2 style="color:green;">Déploiement réussi</h2>
                            <p>Projet: ${projectName}</p>
                            <p>Build: #${buildNumber}</p>
                            <p><a href="${buildUrl}">Voir le build</a></p>
                        """
                    )

                    slackSend(
                        channel: SLACK_CHANNEL,
                        color: 'good',
                        message: """
                        ✅ *Déploiement réussi*
                        Projet: ${projectName}
                        Build: #${buildNumber}
                        <${buildUrl}|Voir le build>
                        """
                    )
                }
            }
        }
    }

    // ===== POST ACTIONS =====
    post {
        always {
            echo 'Nettoyage du workspace...'
            node {
                cleanWs()
            }
        }
        failure {
            echo 'Pipeline en échec'
        }
    }

}
