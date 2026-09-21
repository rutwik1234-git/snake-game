pipeline {
    agent any

    stages {
        stage("Clone Code") {
            steps {
                echo "Cloning The Code"
                git url: "https://github.com/rutwik1234-git/snake-game.git", branch: "main" 
            }
        }

        stage("Code Build & Test") {
            steps {
                echo "Building and testing code"
                sh "docker build -t snake-game ."
            }
        }

        stage("Push Docker image on DockerHub") {
            steps {
                echo "Pushing docker image on DockerHub"
                withCredentials([usernamePassword(credentialsId: "DockerCredentials", passwordVariable: "dockerHubPass", usernameVariable: "dockerHubUser")]) {
                    sh 'echo $dockerHubPass | docker login -u $dockerHubUser --password-stdin'
                    sh 'docker tag snake-game $dockerHubUser/snake-game:latest'
                    sh 'docker push $dockerHubUser/snake-game:latest'
                }
            }
        } 

        stage("Deploy to EKS") {
            steps {
                echo "Deploying application to EKS cluster"
                withCredentials([file(credentialsId: 'k8s-cred', variable: 'KUBECONFIG')]) {
                    sh '''
                        set -e
                        export PATH=$PATH:/usr/local/bin:/usr/bin
                        kubectl --kubeconfig="$KUBECONFIG" create namespace snake \\
                            --dry-run=client -o yaml | \\
                            kubectl --kubeconfig="$KUBECONFIG" apply -f -
                        kubectl --kubeconfig="$KUBECONFIG" apply -f k8s/deployment.yml
                        kubectl --kubeconfig="$KUBECONFIG" apply -f k8s/service.yml
                        kubectl --kubeconfig="$KUBECONFIG" rollout status \\
                            deployment/snake-deployment --namespace=snake
                    '''
                }
            }
        }
    }
}
