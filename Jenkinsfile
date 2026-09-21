pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "kreajith2026/argocd"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                }
            }
        }

        stage('Update K8s Manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                       
                        git clone https://\$GIT_USER:\$GIT_TOKEN@github.com/Antony2026-ai/argocd-test.git
                        
                        sed -i "s|image:.*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" dev/deployment.yaml
                        
                        git add dev/deployment.yaml
                        git commit -m "Update image to build ${BUILD_NUMBER}"
                        git push origin main
                    """
                }
            }
        }
    }
}
