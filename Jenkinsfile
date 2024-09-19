pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    docker.build('your-java-webapp-image:latest')
                }
            }
        }
        stage('Test') {
            steps {
                // Add your test steps here
                echo 'Running tests...'
            }
        }
        stage('Deploy') {
            steps {
                script {
                    // Push Docker image to registry
                    docker.withRegistry('https://your-docker-registry', 'docker-credentials-id') {
                        docker.image('your-java-webapp-image:latest').push('latest')
                    }
                    
                    // Deploy to Kubernetes
                    kubectl.apply('-f java-webapp-deployment.yaml')
                    kubectl.apply('-f java-webapp-service.yaml')
                    kubectl.apply('-f java-webapp-ingress.yaml')
                }
            }
        }
    }
}
