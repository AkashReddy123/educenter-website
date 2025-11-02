pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic to the selected environment after deployment')
    }

    environment {
        IMAGE_NAME = 'akashreddy123/educenter'
        KUBE_NAMESPACE = 'default'
        DOCKER_CREDENTIALS_ID = 'dockerhub-login'
        KUBE_CREDENTIALS_ID = 'kubeconfig'
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/AkashReddy123/educenter-website.git'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    echo "Building Docker image for ${params.DEPLOY_ENV} environment..."
                    bat """
                    docker build -t ${IMAGE_NAME}:${params.DEPLOY_ENV} .
                    """
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat """
                    docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                    docker tag ${IMAGE_NAME}:${params.DEPLOY_ENV} %DOCKER_USER%/${IMAGE_NAME}:${params.DEPLOY_ENV}
                    docker push %DOCKER_USER%/${IMAGE_NAME}:${params.DEPLOY_ENV}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat """
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo Deploying ${params.DEPLOY_ENV} environment...
                    kubectl apply -f educenter-${params.DEPLOY_ENV}-deployment.yaml
                    kubectl apply -f educenter-service.yaml
                    """
                }
            }
        }

        stage('Verify Deployment Health') {
            steps {
                script {
                    echo "🩺 Checking health of ${params.DEPLOY_ENV} environment..."
                    def healthCheck = powershell(
                        script: '''
                        Start-Sleep -Seconds 20
                        try {
                            $response = Invoke-WebRequest -Uri "http://localhost:30082" -UseBasicParsing -TimeoutSec 10
                            if ($response.StatusCode -eq 200) { Write-Output "200" } else { Write-Output "FAIL" }
                        } catch { Write-Output "FAIL" }
                        ''',
                        returnStdout: true
                    ).trim()

                    if (healthCheck != "200") {
                        error "❌ Health check failed for ${params.DEPLOY_ENV}! Aborting deployment."
                    } else {
                        echo "✅ ${params.DEPLOY_ENV} environment is healthy."
                    }
                }
            }
        }

        stage('Switch Traffic to New Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat """
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    echo Switching traffic to ${params.DEPLOY_ENV} environment...
                    kubectl patch svc educenter-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"educenter\\", \\"version\\": \\"${params.DEPLOY_ENV}\\"}}}"
                    """
                }
            }
        }

        stage('Verify Final Status') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    bat """
                    set KUBECONFIG=%KUBECONFIG_FILE%
                    kubectl get pods -l app=educenter -n ${KUBE_NAMESPACE}
                    kubectl get svc educenter-service -n ${KUBE_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Blue-Green deployment successful! Active environment: ${params.DEPLOY_ENV}"
        }
        failure {
            echo "❌ Deployment failed. Check logs and verify the Kubernetes state."
        }
    }
}
