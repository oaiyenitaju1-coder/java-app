pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://sonarqube:9000'
        APP_IMAGE = 'horla1/java-app:latest'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                    echo "CHECKOUT COMPLETE"
                    pwd
                    ls -la
                '''
            }
        }

        stage('Build with Java 17') {
            steps {
                sh '''
                    docker exec java17-builder sh -lc 'rm -rf /workspace && mkdir -p /workspace'
                    docker cp . java17-builder:/workspace
                    docker exec java17-builder sh -lc "
                        cd /workspace &&
                        java -version &&
                        mvn -B clean compile
                    "
                '''
            }
        }

        stage('Test with Java 11') {
            steps {
                sh '''
                    docker exec java11-tester sh -lc 'rm -rf /workspace && mkdir -p /workspace'
                    docker cp . java11-tester:/workspace
                    docker exec java11-tester sh -lc "
                        cd /workspace &&
                        java -version &&
                        mvn -B test
                    "
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        echo "USING JAVA11 SONAR STAGE"
                        grep -n "sonar:sonar" Jenkinsfile || true
                        grep -n "sonar-maven-plugin" Jenkinsfile || true

                        docker exec java11-sonar sh -lc 'rm -rf /workspace && mkdir -p /workspace'
                        docker cp . java11-sonar:/workspace
                        docker exec java11-sonar sh -lc "
                            cd /workspace &&
                            java -version &&
                            mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:4.0.0.4121:sonar \
                              -Dsonar.projectKey=java-app \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_TOKEN}
                        "
                    '''
                }
            }
        }

        stage('Package') {
            steps {
                sh '''
                    docker exec java17-builder sh -lc 'rm -rf /workspace && mkdir -p /workspace'
                    docker cp . java17-builder:/workspace
                    docker exec java17-builder sh -lc "
                        cd /workspace &&
                        mvn -B clean package -DskipTests
                    "
                    docker cp java17-builder:/workspace/target .
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${APP_IMAGE} .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${APP_IMAGE}
                    '''
                }
            }
        }

        stage('Deploy App Container') {
            steps {
                sh '''
                    docker rm -f java-app-test || true
                    docker run -d --name java-app-test -p 8081:8080 ${APP_IMAGE}
                '''
            }
        }
    }

    post {
        always {
            sh '''
                echo "Pipeline finished"
            '''
        }
        success {
            sh '''
                echo "Pipeline completed successfully"
            '''
        }
        failure {
            sh '''
                echo "Pipeline failed"
            '''
        }
    }
}
