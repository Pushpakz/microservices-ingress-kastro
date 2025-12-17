pipeline {
    agent any

    environment {
        DOCKER_HUB_REPO = 'pushpakz/techsolutions-app'
        K8S_CLUSTER_NAME = 'kastro-cluster'
        AWS_REGION = 'us-east-1'
        NAMESPACE = 'default'
        APP_NAME = 'techsolutions'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                git 'https://github.com/Pushpakz/microservices-ingress-kastro.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    env.IMAGE_TAG = env.BUILD_NUMBER
                    sh """
                      docker build -t ${DOCKER_HUB_REPO}:${IMAGE_TAG} .
                      docker tag ${DOCKER_HUB_REPO}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:latest
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo 'Pushing Docker image to DockerHub...'
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh """
                      echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                      docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                      docker push ${DOCKER_HUB_REPO}:latest
                    """
                }
            }
        }

        stage('Configure AWS and Kubectl') {
            steps {
                echo 'Configuring AWS CLI and kubectl...'
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']
                ]) {
                    sh """
                      aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${K8S_CLUSTER_NAME}

                      kubectl get nodes
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying application to Kubernetes...'
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']
                ]) {
                    sh """
                      sed -i 's|pushpakz/techsolutions-app:.*|pushpakz/techsolutions-app:${IMAGE_TAG}|g' k8s/deployment.yaml

                      kubectl apply -n ${NAMESPACE} -f k8s/deployment.yaml
                      kubectl rollout status deployment/${APP_NAME}-deployment -n ${NAMESPACE} --timeout=300s
                      kubectl get pods -n ${NAMESPACE} -l app=${APP_NAME}
                      kubectl get svc -n ${NAMESPACE} ${APP_NAME}-service
                    """
                }
            }
        }

      stage('Deploy Ingress') {
    steps {
        echo 'Deploying Ingress resource...'
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']
        ]) {
            sh """
              kubectl apply -f k8s/ingress.yaml
              sleep 10
              kubectl get ingress ${APP_NAME}-ingress
            """
        }
    }
}


        stage('Get Ingress URL') {
            steps {
                echo 'Fetching Ingress URL...'
                timeout(time: 10, unit: 'MINUTES') {
                    waitUntil {
                        script {
                            def host = sh(
                                script: "kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                                returnStdout: true
                            ).trim()

                            if (host) {
                                env.INGRESS_URL = "http://${host}"
                                echo "Application URL: ${env.INGRESS_URL}"
                                return true
                            }
                            return false
                        }
                    }
                }

                sh """
                  curl -I ${INGRESS_URL}/ || true
                  curl -I ${INGRESS_URL}/about || true
                  curl -I ${INGRESS_URL}/services || true
                  curl -I ${INGRESS_URL}/contact || true
                """
            }
        }
    }

    post {
        always {
            echo 'Cleaning up Docker images...'
            sh """
              docker rmi ${DOCKER_HUB_REPO}:${IMAGE_TAG} || true
              docker rmi ${DOCKER_HUB_REPO}:latest || true
            """
        }

        success {
            echo '====================================='
            echo ' PIPELINE COMPLETED SUCCESSFULLY '
            echo ' Application URL: ' + env.INGRESS_URL
            echo '====================================='
        }

        failure {
            echo 'Pipeline failed! Please check the logs.'
        }
    }
}

