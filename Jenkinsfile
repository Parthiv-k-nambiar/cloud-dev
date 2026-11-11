
pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'YOUR_DOCKERHUB_USERNAME'
        IMAGE_NAME = 'vle7-web'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    test -f app/index.html
                    test -f app/Dockerfile
                    test -f k8s/deployment.yaml
                    test -f k8s/service.yaml
                    echo "All project files found successfully"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE = "${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
                sh '''
                    docker build -f app/Dockerfile -t "$IMAGE" .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login \
                            -u "$DH_USER" --password-stdin

                        docker push "$IMAGE"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig',
                         variable: 'KUBECONFIG')
                ]) {
                    sh '''
                        sed "s|DOCKERHUB_USERNAME/vle7-web:latest|$IMAGE|g" \
                            k8s/deployment.yaml > deployment-temp.yaml

                        kubectl apply -f deployment-temp.yaml
                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status deployment/vle7-web \
                            --timeout=180s
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig',
                         variable: 'KUBECONFIG')
                ]) {
                    sh '''
                        kubectl get deployments
                        kubectl get pods
                        kubectl get services
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'VLE 7 CI/CD Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }
    }
}
