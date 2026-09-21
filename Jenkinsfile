pipeline {
    agent any

    environment {
        APP_NAME = 'snake-game'
        K8S_NAMESPACE = 'snake'
    }

    stages {
        stage("Code Build & Test") {
            steps {
                echo "Building Docker image version ${BUILD_NUMBER}..."
                sh """
                    docker build -t ${APP_NAME}:${BUILD_NUMBER} .
                    docker tag ${APP_NAME}:${BUILD_NUMBER} ${APP_NAME}:latest
                """
            }
        }

        stage("Push Docker Image") {
            steps {
                echo "Pushing Docker image to DockerHub..."
                withCredentials([usernamePassword(credentialsId: "DockerCredentials", passwordVariable: "dockerHubPass", usernameVariable: "dockerHubUser")]) {
                    sh '''
                        echo "$dockerHubPass" | docker login -u "$dockerHubUser" --password-stdin
                        
                        docker tag ${APP_NAME}:${BUILD_NUMBER} $dockerHubUser/${APP_NAME}:${BUILD_NUMBER}
                        docker tag ${APP_NAME}:${BUILD_NUMBER} $dockerHubUser/${APP_NAME}:latest
                        
                        docker push $dockerHubUser/${APP_NAME}:${BUILD_NUMBER}
                        docker push $dockerHubUser/${APP_NAME}:latest
                    '''
                }
            }
        } 

        stage("Deploy to Kubernetes") {
            steps {
                echo "Deploying application to EKS cluster..."
                withCredentials([file(credentialsId: 'k8s-cred', variable: 'KUBECONFIG')]) {
                    sh '''
                        set -e
                        export PATH=$PATH:/usr/local/bin:/usr/bin

                        # Ensure namespace exists
                        kubectl --kubeconfig="$KUBECONFIG" create namespace ${K8S_NAMESPACE} \
                            --dry-run=client -o yaml | \
                            kubectl --kubeconfig="$KUBECONFIG" apply -f -

                        # Apply manifests
                        kubectl --kubeconfig="$KUBECONFIG" apply -f k8s/deployment.yml -n ${K8S_NAMESPACE}
                        kubectl --kubeconfig="$KUBECONFIG" apply -f k8s/service.yml -n ${K8S_NAMESPACE}

                        # Force rollout restart to use latest image version
                        kubectl --kubeconfig="$KUBECONFIG" rollout restart deployment/snake-deployment -n ${K8S_NAMESPACE}

                        # Monitor deployment progress
                        kubectl --kubeconfig="$KUBECONFIG" rollout status deployment/snake-deployment -n ${K8S_NAMESPACE} --timeout=180s
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up local build artifacts..."
            sh '''
                docker rmi ${APP_NAME}:${BUILD_NUMBER} || true
            '''
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
