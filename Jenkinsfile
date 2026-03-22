pipeline {
    agent any

    environment {
        APP_NAME        = 'java-app'
        DOCKERHUB_USER  = 'horla1'
        IMAGE_NAME      = "${horla1}/java-app"
        SONAR_HOST_URL  = 'http://sonarqube:9000'
    }

    options {
        timestamps()
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build with Java 17') {
            steps {
                sh '''
                    docker exec java17-builder sh -lc 'rm -rf /workspace && mkdir -p /workspace'
                    docker cp . java17-builder:/workspace
                    docker exec java17-builder sh -lc '
                        cd /workspace &&
                        mvn -B clean compile
                    '
                '''
            }
        }

        stage('Run Unit Tests with Java 11') {
            steps {
                sh '''
                    docker exec java11-tester sh -lc 'rm -rf /workspace && mkdir -p /workspace'
                    docker cp . java11-tester:/workspace
                    docker exec java11-tester sh -lc '
                        cd /workspace &&
                        mvn -B test
                    '
                '''
            }
        }

        stage('SonarQube Analysis with Java 8') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        docker exec java8-analyzer sh -lc 'rm -rf /workspace && mkdir -p /workspace'
                        docker cp . java8-analyzer:/workspace
                        docker exec java8-analyzer sh -lc "
                            cd /workspace &&
                            mvn -B sonar:sonar \
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
                sh '''
                    sed "s|YOUR_DOCKER_HUB_USERNAME/java-app:latest|${IMAGE_NAME}:${BUILD_NUMBER}|g" deployment.yaml > deployment.rendered.yaml
                    kubectl apply -f deployment.rendered.yaml
                    kubectl rollout status deployment/java-app
                    kubectl get pods
                    kubectl get svc
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
        always {
            sh '''
                docker logout || true
            '''
        }
    }
}
