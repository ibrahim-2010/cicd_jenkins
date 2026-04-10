pipeline {
    agent any
    tools {
        maven 'mvn'
        jdk   'openjdk'
    }
    environment {
        DOCKER_CREDS = credentials('jenkins_credentials')
    }
    stages {
        stage('Cleanup') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/ibrahim-2010/cicd_jenkins.git'
            }
        }
        stage('Unit Testing') {
            steps {
                sh 'mvn test'
            }
        }
        stage('DockerBuild') {
            steps {
                sh "docker build -t ibj2010/nginx:${BUILD_NUMBER} ."
            }
        }
        stage('DockerPush') {
            steps {
                sh '''
                    echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
                '''
                sh "docker push ibj2010/nginx:${BUILD_NUMBER}"
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    // Update kubeconfig to connect to EKS cluster
                    sh 'aws eks update-kubeconfig --name ibra --region us-east-2'
                    // Replace placeholder with actual image tag
                    sh "sed -i 's|IMAGE_PLACEHOLDER|ibj2010/nginx:${BUILD_NUMBER}|g' k8s/deployment.yaml"
                    // Apply the Kubernetes manifests
                    sh 'kubectl apply -f k8s/deployment.yaml'
                    // Verify rollout completes successfully
                    sh 'kubectl rollout status deployment/ibra-app --timeout=120s'
                }
            }
        }
    }
}
