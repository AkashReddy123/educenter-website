pipeline {
    agent any

    parameters {
        choice(
            name: 'DEPLOY_VERSION',
            choices: ['auto', 'blue', 'green'],
            description: 'Select which version to deploy (auto will detect the opposite of the active one)'
        )
    }

    environment {
        DOCKER_IMAGE = "educenter"
        BLUE_TAG = "blue"
        GREEN_TAG = "green"
        DOCKER_CREDENTIALS_ID = "dockerhub-login"
        KUBE_CREDENTIALS_ID = "kubeconfig"
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
                    def detectedVersion = bat(
                        script: '''
                        @echo off
                        setlocal enabledelayedexpansion
                        for /f "tokens=* usebackq" %%A in (`kubectl get svc educenter-service -o jsonpath="{.spec.selector.version}" 2^>nul`) do set VERSION=%%A
                        if not defined VERSION set VERSION=none
                        echo !VERSION!
                        endlocal
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "🔍 Currently active version in cluster: ${detectedVersion}"

                    if (params.DEPLOY_VERSION != 'auto') {
                        env.NEW_VERSION = params.DEPLOY_VERSION
                        echo "🚀 Manual selection: deploying ${env.NEW_VERSION} version."
                    } else {
                        if (detectedVersion == "blue") {
                            env.NEW_VERSION = "green"
                            echo "🟦 Blue is active → Auto-switching to Green"
                        } else if (detectedVersion == "green") {
                            env.NEW_VERSION = "blue"
                            echo "🟩 Green is active → Auto-switching to Blue"
                        } else {
                            env.NEW_VERSION = "blue"
                            echo "⚙️ No active version detected → Deploying Blue (first deployment)"
                        }
                    }

                    // Log final target
                    echo "🎯 Target deployment version: ${env.NEW_VERSION}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                echo 🏗️ Building Docker image %DOCKER_IMAGE%:%NEW_VERSION% ...
                docker build -t %DOCKER_IMAGE%:%NEW_VERSION% .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat '''
                    echo 🔐 Logging in to Docker Hub...
                    docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                    docker tag %DOCKER_IMAGE%:%NEW_VERSION% %DOCKER_USER%/%DOCKER_IMAGE%:%NEW_VERSION%
                    echo 📤 Pushing image to Docker Hub...
                    docker push %DOCKER_USER%/%DOCKER_IMAGE%:%NEW_VERSION%
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat '''
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo 🚀 Deploying %NEW_VERSION% version to Kubernetes...
                    kubectl apply -f educenter-%NEW_VERSION%-deployment.yaml
                    kubectl apply -f educenter-service.yaml
                    '''
                }
            }
        }

        stage('Health Check Inside Cluster') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat '''
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo 🩺 Checking pod health inside cluster for %NEW_VERSION%...
                    setlocal enabledelayedexpansion
                    for /L %%i in (1,1,10) do (
                        echo --- Attempt %%i ---
                        kubectl get pods -l app=educenter,version=%NEW_VERSION% -o custom-columns="NAME:.metadata.name,STATUS:.status.phase,READY:.status.containerStatuses[*].ready" -n default
                        kubectl get pods -l app=educenter,version=%NEW_VERSION% -o jsonpath="{.items[*].status.containerStatuses[*].ready}" > ready.txt
                        set /p READY=<ready.txt
                        if "!READY!"=="true" (
                            echo ✅ Pods for %NEW_VERSION% are READY!
                            exit /b 0
                        )
                        echo ⏳ Waiting for pods to become ready... (%%i/10)
                        timeout /t 10 >nul
                    )
                    echo ❌ Pods for %NEW_VERSION% failed to become ready in time.
                    exit /b 1
                    endlocal
                    '''
                }
            }
        }

        stage('Switch Service to New Version') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat '''
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo 🔁 Switching service to %NEW_VERSION% version...
                    kubectl patch svc educenter-service --type merge -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"educenter\\", \\"version\\": \\"%NEW_VERSION%\\"}}}"
                    echo ✅ Traffic switched successfully to %NEW_VERSION%!

                    echo 🌐 Checking currently active service version...
                    kubectl get svc educenter-service -o=jsonpath="{.spec.selector.version}"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Blue-Green Deployment Successful! Active version: ${env.NEW_VERSION}"
        }
        failure {
            echo "❌ Deployment Failed. Please check Jenkins logs or rollback manually."
        }
    }
}
