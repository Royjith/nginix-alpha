pipeline {
    agent { label 'node-1' }  // Use node-1 agent for execution

    environment {
        DOCKER_IMAGE = 'my-app'               // Docker image name
        DOCKER_TAG = 'version-1.10'              // Docker tag
        DOCKER_HUB_REPO = 'royjith/cube'     // Docker Hub repository
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub' // Docker Hub credentials ID
        KUBE_CONFIG = '/tmp/kubeconfig'       // Path to kubeconfig file
        TEST_NAMESPACE = 'test'               // Kubernetes namespace for test
        PROD_NAMESPACE = 'prod'               // Kubernetes namespace for prod
    }

    stages {
        // Checkout the source code from the Git repository
        stage('Checkout') {
            steps {
                echo 'Checking out code from Git...'
                git branch: 'main', url: 'https://github.com/Royjith/nginix-alpha.git'
            }
        }

        // Build Docker image and tag it
        stage('Build Docker Image') {
            steps {
                script {
                    def tag = "${DOCKER_TAG}"
                    echo "Building Docker image with tag: ${tag}..."
                    try {
                        sh """
                            docker rmi -f ${DOCKER_HUB_REPO}:${tag} || true
                            docker system prune -f
                        """
                        sh "docker build -f Dockerfile --no-cache -t ${DOCKER_HUB_REPO}:${tag} ."
                        sh "docker tag ${DOCKER_HUB_REPO}:${tag} ${DOCKER_HUB_REPO}:${tag}"
                    } catch (Exception e) {
                        error "Docker build failed: ${e.message}"
                    }
                }
            }
        }

        // Run Trivy security scan for vulnerabilities
        stage('Trivy Scan') {
            steps {
                script {
                    def scanResult = sh(script: "trivy image ${DOCKER_HUB_REPO}:${DOCKER_TAG}", returnStatus: true)
                    if (scanResult != 0) {
                        error 'Trivy scan found vulnerabilities in the Docker image!'
                    } else {
                        echo 'Trivy scan passed: No vulnerabilities found.'
                    }
                }
            }
        }

        // Push Docker image to Docker Hub
        stage('Push Image to DockerHub') {
            steps {
                input message: 'Approve Deployment?', ok: 'Deploy'
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_HUB_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        sh "docker push ${DOCKER_HUB_REPO}:${DOCKER_TAG}"
                    }
                }
            }
        }

        // Deploy Docker image to Kubernetes Test namespace
        stage('Deploy to Kubernetes - Test') {
            steps {
                input message: 'Approve Kubernetes Deployment to Test?', ok: 'Deploy'
                script {
                    withCredentials([file(credentialsId: 'pikube', variable: 'KUBECONFIG_FILE')]) {
                        def deploymentFile = 'deployment.yaml'

                        // Update the image and namespace in deployment.yaml
                        sh """
                            sed -i 's|image: .*|image: ${DOCKER_HUB_REPO}:${DOCKER_TAG}|g' ${deploymentFile}
                            sed -i 's|namespace: .*|namespace: ${TEST_NAMESPACE}|g' ${deploymentFile}
                        """

                        // Apply the deployment to Kubernetes Test namespace
                        sh """
                            export KUBECONFIG=$KUBECONFIG_FILE
                            kubectl apply -f ${deploymentFile} --namespace=${TEST_NAMESPACE}
                        """
                    }
                }
            }
        }

        // Deploy Docker image to Kubernetes Prod namespace
        stage('Deploy to Kubernetes - Prod') {
            steps {
                input message: 'Approve Kubernetes Deployment to Prod?', ok: 'Deploy'
                script {
                    withCredentials([file(credentialsId: 'pikube', variable: 'KUBECONFIG_FILE')]) {
                        def deploymentFile = 'deployment.yaml'

                        // Update the image and namespace in deployment.yaml
                        sh """
                            sed -i 's|image: .*|image: ${DOCKER_HUB_REPO}:${DOCKER_TAG}|g' ${deploymentFile}
                            sed -i 's|namespace: .*|namespace: ${PROD_NAMESPACE}|g' ${deploymentFile}
                        """

                        // Apply the deployment to Kubernetes Prod namespace
                        sh """
                            export KUBECONFIG=$KUBECONFIG_FILE
                            kubectl apply -f ${deploymentFile} --namespace=${PROD_NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // Clean workspace after pipeline execution
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
