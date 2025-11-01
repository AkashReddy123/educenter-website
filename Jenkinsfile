pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "educenter"
        BLUE_TAG = "blue"
        GREEN_TAG = "green"
        DOCKER_HUB_USER = "balaakashreddyy"
        K8S_NAMESPACE = "default"
    }

    stages {
        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/AkashReddy123/educenter-website.git'
            }
        }

        stage('Set Active Color') {
            steps {
                script {
                    def activeColor = bat(script: 'kubectl get svc educenter-service -o jsonpath="{.spec.selector.color}" 2>nul || echo blue', returnStdout: true).trim()
                    if (activeColor == "blue") {
                        env.NEW_COLOR = "green"
                        echo "Blue is active → Deploying Green"
                    } else {
                        env.NEW_COLOR = "blue"
                        echo "Green is active → Deploying Blue"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat """
                docker build -t %DOCKER_HUB_USER%/%DOCKER_IMAGE%:%NEW_COLOR% .
                docker tag %DOCKER_HUB_USER%/%DOCKER_IMAGE%:%NEW_COLOR% %DOCKER_HUB_USER%/%DOCKER_IMAGE%:latest
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([string(credentialsId: 'docker-pass', variable: 'DOCKER_PASS')]) {
                    bat """
                    echo %DOCKER_PASS% | docker login -u %DOCKER_HUB_USER% --password-stdin
                    docker push %DOCKER_HUB_USER%/%DOCKER_IMAGE%:%NEW_COLOR%

                    REM ✅ Check if latest digest already exists on Docker Hub
                    for /f "tokens=* usebackq" %%A in (`docker inspect --format="{{.Id}}" %DOCKER_HUB_USER%/%DOCKER_IMAGE%:%NEW_COLOR%`) do set LOCAL_DIGEST=%%A
                    for /f "tokens=* usebackq" %%B in (`docker inspect --format="{{.Id}}" %DOCKER_HUB_USER%/%DOCKER_IMAGE%:latest 2^>nul`) do set REMOTE_DIGEST=%%B

                    if "%LOCAL_DIGEST%"=="%REMOTE_DIGEST%" (
                        echo Skipping push for :latest (digest unchanged)
                    ) else (
                        echo Pushing :latest tag...
                        docker push %DOCKER_HUB_USER%/%DOCKER_IMAGE%:latest
                    )
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                bat """
                kubectl set image deployment/educenter-%NEW_COLOR% educenter-container=%DOCKER_HUB_USER%/%DOCKER_IMAGE%:%NEW_COLOR% -n %K8S_NAMESPACE% ^
                || kubectl apply -f educenter-%NEW_COLOR%-deployment.yaml

                kubectl apply -f educenter-service.yaml
                """
            }
        }

        stage('Switch Service to New Version') {
            steps {
                bat """
                kubectl patch svc educenter-service -p "{\"spec\":{\"selector\":{\"app\":\"educenter-%NEW_COLOR%\",\"color\":\"%NEW_COLOR%\"}}}"
                """
                echo "✅ Service switched to ${env.NEW_COLOR} deployment successfully!"
            }
        }
    }
}
