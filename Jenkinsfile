pipeline {
    agent any

    environment {
        IMAGE_NAME = 'horla1/java-app'
        SONAR_HOST_URL = 'http://sonarqube:9000'
        GITOPS_REPO = 'https://github.com/oaiyenitaju1-coder/java-app-gitops.git'
        GITOPS_BRANCH = 'main'
        APP_NAME = 'java-app'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/oaiyenitaju1-coder/java-app.git'
            }
        }

        stage('Ensure Required Containers Are Running') {
            steps {
                sh '''
                    docker start java11-tester || true
                    docker start java11-sonar || true
                    docker start sonarqube || true
                    docker start minikube || true

                    docker ps
                '''
            }
        }

        stage('Build with Maven (Java 11)') {
            steps {
                sh '''
                    docker exec java11-tester sh -lc '
                        rm -rf /workspace &&
                        mkdir -p /workspace
                    '

                    docker cp . java11-tester:/workspace

                    docker exec java11-tester sh -lc '
                        cd /workspace &&
                        mvn -B clean test
                    '
                '''
            }
        }

        stage('SonarQube Analysis (Java 11)') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        docker exec java11-sonar sh -lc '
                            rm -rf /workspace &&
                            mkdir -p /workspace
                        '

                        docker cp . java11-sonar:/workspace

                        docker exec java11-sonar sh -lc "
                            cd /workspace &&
                            mvn -B clean verify sonar:sonar \
                              -Dsonar.projectKey=java-app \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_TOKEN}
                        "
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    trivy image --exit-code 0 --severity HIGH,CRITICAL ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        echo "Current context:"
                        kubectl config current-context

                        echo "Checking cluster access..."
                        kubectl get nodes

                        kubectl apply -f deployment.yaml

                        kubectl set image deployment/java-app \
                            java-app=${IMAGE_NAME}:${BUILD_NUMBER}

                        kubectl rollout status deployment/java-app

                        kubectl get pods
                        kubectl get svc
                    '''
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {
                    sh '''
                        rm -rf gitops-repo

                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/oaiyenitaju1-coder/java-app-gitops.git gitops-repo

                        cd gitops-repo

                        sed -i.bak "s|image: .*|image: ${IMAGE_NAME}:${BUILD_NUMBER}|g" deployment.yaml || true
                        rm -f deployment.yaml.bak

                        git config user.name "Jenkins"
                        git config user.email "jenkins@local"

                        git add .
                        git commit -m "Update java-app image to ${BUILD_NUMBER}" || true
                        git push origin ${GITOPS_BRANCH}
                    '''
                }
            }
        }

        stage('Verify Argo CD Sync') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'argocd-creds',
                    usernameVariable: 'ARGOCD_USER',
                    passwordVariable: 'ARGOCD_PASS'
                )]) {
                    sh '''
                        argocd login 192.168.49.4:30080 \
                          --username ${ARGOCD_USER} \
                          --password ${ARGOCD_PASS} \
                          --insecure

                        argocd app sync ${APP_NAME}
                        argocd app wait ${APP_NAME} --health --sync
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
