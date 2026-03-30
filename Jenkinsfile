pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'your-dockerhub-username/cicd-hello-world:latest'
        SONAR_HOST = 'http://sonarqube:9000'
        DOCKER_USERNAME = credentials('dockerhub-creds')
        DOCKER_PASSWORD = credentials('dockerhub-creds')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/oaiyenitaju1-coder/java-app.git'
            }
        }

        stage('Build (Java 17)') {
            steps {
                sh '''
                    docker start java17-builder || true

                    docker exec java17-builder bash -c "
                        rm -rf /app &&
                        mkdir -p /app
                    "
                    docker cp . java17-builder:/app
                    docker exec java17-builder bash -c "
                        cd /app &&
                        ./mvnw clean package -DskipTests || mvn clean package -DskipTests
                    "
                '''
            }
        }

        stage('Run Unit Tests (Java 11)') {
            steps {
                sh '''
                    docker start java11-tester || true

                    docker exec java11-tester bash -c "
                        rm -rf /app &&
                        mkdir -p /app
                    "
                    docker cp . java11-tester:/app
                    docker exec java11-tester bash -c "
                        cd /app &&
                        ./mvnw test || mvn test
                    "
                '''
            }
        }

        stage('Code Analysis (Java 8)') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        docker start java11-tester || true
                        docker start sonarqube || true

                        docker exec java11-tester bash -c "
                            rm -rf /app &&
                            mkdir -p /app
                        "
                        docker cp . java11-tester:/app
                        docker exec java11-tester bash -c "
                            cd /app &&
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:4.0.0.4121:sonar \
                                -Dsonar.host.url=${SONAR_HOST} \
                                -Dsonar.login=${SONAR_TOKEN}
                        "
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t horla1/java-app:latest .
                '''
            }
        }

        stage('Security Scan (Trivy)') {
            agent {
                docker {
                    image 'ghcr.io/aquasecurity/trivy:latest'
                    args "-u 0 -v /var/run/docker.sock:/var/run/docker.sock --entrypoint=''"
                }
            }
            steps {
                script {
                    sh "trivy image --cache-dir .trivycache --severity HIGH,CRITICAL horla1/java-app:latest"
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push horla1/java-app:latest
                    '''
                }
            }
        }

        stage('Deploy with Helm') {
            agent {
                docker {
                    image 'alpine/helm:latest'
                    args "-v /var/run/docker.sock:/var/run/docker.sock --entrypoint=''"
                }
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        helm upgrade --install my-release ./helm/hello-world \
                            --set image.tag=latest \
                            --kube-insecure-skip-tls-verify
                    '''
                }
            }
        }

        stage('Check Pods') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        kubectl get pods --insecure-skip-tls-verify
                    '''
                }
            }
        }

    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed — check the logs above.'
        }
    }
}
