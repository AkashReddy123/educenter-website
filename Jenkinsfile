pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "educenter"
        BLUE_TAG = "blue"
        GREEN_TAG = "green"
        DOCKER_HUB_USER = "balaakashreddyy"   // your Docker Hub username
        K8S_NAMESPACE = "default"
    }

    stages {
        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/AkashReddyy/educenter-website.git'
            }
        }

        stage('Set Active Color') {
            steps {
                script {
                    def activeColor = sh(script: "kubectl get svc educenter-service -o=jsonpath='{.spec.selector.color}' || echo blue", returnStdout: true).trim()
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
                script {
                    sh """
                    docker build -t ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:${NEW_COLOR} .
                    docker tag ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:${NEW_COLOR} ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                // ✅ Works with Docker Hub username + password
                withCredentials([usernamePassword(credentialsId: 'dockerhub-login', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:${NEW_COLOR}
                    docker push ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                    kubectl set image deployment/educenter-${NEW_COLOR} educenter-container=${DOCKER_HUB_USER}/${DOCKER_IMAGE}:${NEW_COLOR} -n ${K8S_NAMESPACE} \
                    || kubectl apply -f educenter-${NEW_COLOR}-deployment.yaml

                    kubectl apply -f educenter-service.yaml
                    """
                }
            }
        }

        stage('Switch Service to New Version') {
            steps {
                script {
                    sh """
                    kubectl patch svc educenter-service -p '{"spec":{"selector":{"app":"educenter-${NEW_COLOR}","color":"${NEW_COLOR}"}}}'
                    """
                    echo "✅ Service switched to ${NEW_COLOR} deployment successfully!"
                }
            }
        }
    }
}
