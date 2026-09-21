pipeline {
    agent any

    environment {
        DOCKER_IMAGE     = "kreajith2026/argocd"
        DEPLOYMENT_NAME  = "my-python-app"                              // ⭐ change per service
        GITOPS_REPO      = "github.com/Antony2026-ai/argocd-test.git"   // token illama
        MANIFEST_PATH    = "dev/deployment.yaml"
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
    }

    
    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "${DOCKER_IMAGE}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Update K8s Manifest') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        set -e

                        command -v yq >/dev/null 2>&1 || { echo "yq not installed"; exit 1; }

                        rm -rf gitops-tmp
                        git clone https://${GIT_USER}:${GIT_TOKEN}@${GITOPS_REPO} gitops-tmp
                        cd gitops-tmp

                        # ⭐ Dynamic — only this deployment image update
                        yq -i '
                          (select(.kind == "Deployment" and .metadata.name == env(DEPLOYMENT_NAME))
                           .spec.template.spec.containers[0].image)
                          = env(DOCKER_IMAGE) + ":" + env(IMAGE_TAG)
                        ' "${MANIFEST_PATH}"

                        git config user.email "jenkins@ci.com"
                        git config user.name  "Jenkins CI"
                        git add "${MANIFEST_PATH}"

                        if git diff --cached --quiet; then
                            echo "No changes to commit"
                        else
                            git commit -m "chore(${DEPLOYMENT_NAME}): image ${IMAGE_TAG}"
                            git push origin main
                            echo "✅ Updated ${DEPLOYMENT_NAME} → ${IMAGE_TAG}"
                        fi
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'rm -rf gitops-tmp || true'
            }
        }
    }

    post {
        always {
            sh 'docker logout || true; rm -rf gitops-tmp || true'
        }
        success {
            echo "✅ ${DEPLOYMENT_NAME}:${IMAGE_TAG} deployed via ArgoCD"
        }
        failure {
            echo "❌ ${DEPLOYMENT_NAME} build ${IMAGE_TAG} failed"
        }
    }
}
