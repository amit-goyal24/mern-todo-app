pipeline {
    agent any

    tools {
        nodejs 'Node18'
    }

    environment {
        SONAR_TOKEN = credentials('Sonar_token')
        NEXUS_CREDS = credentials('nexus-creds')
        NEXUS_URL = "http://localhost:8081"
        NEXUS_REPO = "raw-release"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out code from GitHub"
                checkout scm
            }
        }

        stage('Install Backend Deps') {
            steps {
                dir('TODO/todo_backend') {
                    echo "Installing backend dependencies"
                    bat 'npm ci'
                }
            }
        }

        stage('Frontend Install & Build') {
            steps {
                dir('TODO/todo_frontend') {
                    withEnv(["PATH+NODEJS=${tool 'Node18'}"]) {
                        bat 'npm install'
                        bat 'npm run build'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('MySonarQubeServer') {
                    bat """
                        ${tool('SonarScanner')}\\bin\\sonar-scanner.bat ^
                          -Dsonar.projectKey=mern-todo-app ^
                          -Dsonar.sources=TODO/todo_backend,TODO/todo_frontend ^
                          -Dsonar.host.url=http://localhost:9000 ^
                          -Dsonar.login=%SONAR_TOKEN% ^
                          -Dsonar.exclusions=**/coverage/**,**/lcov-report/**,**/node_modules/**
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline stopped: Quality Gate FAILED"
                        }
                    }
                }
            }
        }

        stage('Package Artifacts') {
            steps {
                echo "Packaging backend and frontend"

                bat '''
                    if exist artifacts rmdir /s /q artifacts
                    mkdir artifacts
                    powershell -Command "Compress-Archive -Path TODO\\todo_backend\\* -DestinationPath artifacts\\backend-%BUILD_NUMBER%.zip"
                    powershell -Command "Compress-Archive -Path TODO\\todo_frontend\\build\\* -DestinationPath artifacts\\frontend-%BUILD_NUMBER%.zip"
                '''

                archiveArtifacts artifacts: 'artifacts/*', fingerprint: true
            }
        }

        stage('Upload to Nexus') {
            steps {
                echo "Uploading artifacts to Nexus Repository"
                bat """
                    curl -v -u %NEXUS_CREDS_USR%:%NEXUS_CREDS_PSW% --upload-file artifacts/backend-%BUILD_NUMBER%.zip %NEXUS_URL%/repository/%NEXUS_REPO%/backend-%BUILD_NUMBER%.zip
                    curl -v -u %NEXUS_CREDS_USR%:%NEXUS_CREDS_PSW% --upload-file artifacts/frontend-%BUILD_NUMBER%.zip %NEXUS_URL%/repository/%NEXUS_REPO%/frontend-%BUILD_NUMBER%.zip
                """
            }
        }
    }

    post {
        always {
            emailext(
                subject: "Pipeline Status: ${currentBuild.result}",
                body: """<html>
                    <body>
                        <p>Build Status: ${currentBuild.result}</p>
                        <p>Build Number: ${currentBuild.number}</p>
                        <p>Check the <a href="${env.BUILD_URL}">console output</a></p>
                    </body>
                </html>""",
                to: 'amitpvt0150@gmail.com',
                mimeType: 'text/html'
            )
        }
    }
}
