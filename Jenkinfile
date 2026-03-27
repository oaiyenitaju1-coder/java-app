pipeline {
    agent any

    environment {
        IMAGE_NAME = 'horla1/java-app'
        SONAR_HOST_URL = 'http://sonarqube:9000'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/oaiyenitaju1-coder/java-app.git'
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
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
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
