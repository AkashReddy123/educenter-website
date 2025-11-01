pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "educenter"
        BLUE_TAG = "blue"
        GREEN_TAG = "green"
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"
        KUBE_CREDENTIALS_ID = "kubeconfig"
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
                    def activeColor = bat(
                        script: '''
                        @echo off
                        setlocal enabledelayedexpansion
                        for /f "tokens=* usebackq" %%A in (`kubectl get svc educenter-service -o jsonpath="{.spec.selector.color}" 2^>nul`) do set COLOR=%%A
                        if not defined COLOR set COLOR=blue
                        echo !COLOR!
                        endlocal
                        ''',
                        returnStdout: true
                    ).trim()

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
                bat '''
                docker build -t %DOCKER_IMAGE%:%NEW_COLOR% .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat '''
                    docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                    docker tag %DOCKER_IMAGE%:%NEW_COLOR% %DOCKER_USER%/%DOCKER_IMAGE%:%NEW_COLOR%
                    docker push %DOCKER_USER%/%DOCKER_IMAGE%:%NEW_COLOR%
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat '''
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo Deploying %NEW_COLOR% version to Kubernetes...
                    kubectl apply -f educenter-%NEW_COLOR%-deployment.yaml
                    kubectl apply -f educenter-service.yaml
                    '''
                }
            }
        }

        stage('Switch Service to New Version') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat '''
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo Switching service to %NEW_COLOR% version...
                    kubectl patch svc educenter-service -p "{\"spec\": {\"selector\": {\"app\": \"educenter\", \"color\": \"%NEW_COLOR%\"}}}"
                    '''
                }
            }
        }
    }
}
