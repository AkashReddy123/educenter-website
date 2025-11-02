pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "educenter"
        BLUE_TAG = "blue"
        GREEN_TAG = "green"
        DOCKER_CREDENTIALS_ID = "dockerhub-login"
        KUBE_CREDENTIALS_ID = "kubeconfig"
        DOCKER_REPO = "balaakashreddyy"
    }

    stages {

        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/AkashReddy123/educenter-website.git'
            }
        }

        stage('Set Active Version') {
            steps {
                script {
                    def activeVersion = bat(
                        script: '''
                        @echo off
                        setlocal enabledelayedexpansion
                        for /f "tokens=* usebackq" %%A in (`kubectl get svc educenter-service -o jsonpath="{.spec.selector.version}" 2^>nul`) do set VERSION=%%A
                        if not defined VERSION set VERSION=blue
                        echo !VERSION!
                        endlocal
                        ''',
                        returnStdout: true
                    ).trim()

                    if (activeVersion == "blue") {
                        env.NEW_VERSION = "green"
                        echo "🟦 Blue is active → Deploying Green"
                    } else {
                        env.NEW_VERSION = "blue"
                        echo "🟩 Green is active → Deploying Blue"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                echo Building Docker image %DOCKER_REPO%/%DOCKER_IMAGE%:%NEW_VERSION% ...
                docker build -t %DOCKER_REPO%/%DOCKER_IMAGE%:%NEW_VERSION% .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat '''
                    echo Logging in to Docker Hub...
                    docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                    echo Pushing image to Docker Hub...
                    docker push %DOCKER_REPO%/%DOCKER_IMAGE%:%NEW_VERSION%
                    docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat '''
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo Deploying %NEW_VERSION% version to Kubernetes...
                    kubectl apply -f educenter-%NEW_VERSION%-deployment.yaml
                    kubectl apply -f educenter-service.yaml
                    '''
                }
            }
        }

        stage('Health Check Before Switch') {
            steps {
                script {
                    echo "🩺 Checking if ${env.NEW_VERSION} deployment is healthy..."
                    def healthCheck = powershell(
                        script: '''
                        Start-Sleep -Seconds 25
                        try {
                            $response = Invoke-WebRequest -Uri "http://localhost:30082" -UseBasicParsing -TimeoutSec 10
                            if ($response.StatusCode -eq 200) { Write-Output "200" } else { Write-Output "FAIL" }
                        } catch { Write-Output "FAIL" }
                        ''',
                        returnStdout: true
                    ).trim()

                    if (healthCheck != "200") {
                        error "❌ Health check failed for ${env.NEW_VERSION} version! Aborting deployment."
                    } else {
                        echo "✅ Health check passed for ${env.NEW_VERSION} version."
                    }
                }
            }
        }

        stage('Switch Service to New Version') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat '''
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo Switching service to %NEW_VERSION% version...
                    kubectl patch svc educenter-service --type merge -p "{\"spec\": {\"selector\": {\"app\": \"educenter\", \"version\": \"%NEW_VERSION%\"}}}"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Blue-Green Deployment Successful! New active version: ${env.NEW_VERSION}"
        }
        failure {
            echo "❌ Deployment Failed. Please check logs or rollback manually."
        }
    }
}
