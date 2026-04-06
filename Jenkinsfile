pipeline {
    agent any

    environment {
        IMAGE_NAME = 'horla1/java-app:latest'
        SONAR_HOST = 'http://sonarqube:9000'
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
                    docker build -t ${IMAGE_NAME} .
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
                retry(3) {
                    sh '''
                      trivy image \
                        --db-repository ghcr.io/aquasecurity/trivy-db:2 \
                        --java-db-repository ghcr.io/aquasecurity/trivy-java-db:1 \
                        --cache-dir .trivycache \
                        --timeout 20m \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        $IMAGE_NAME
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${IMAGE_NAME}
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
                        helm upgrade --install my-release ./helm/java-app \
                            --set image.repository=horla1/java-app \
                            --set image.tag=latest \
                            --set service.nodePort=30081 \
                            --kube-insecure-skip-tls-verify
                    '''
                }
            }
        }

        stage('Deploy with Argo CD') {
            agent {
                docker {
                    image 'quay.io/argoproj/argocd:latest'
                    args "--entrypoint=''"
                }
            }
            steps {
                withCredentials([string(credentialsId: 'argocd-creds', variable: 'ARGOCD_AUTH_TOKEN')]) {
                    sh '''
                        argocd login host.docker.internal:30008 \
                            --auth-token $ARGOCD_AUTH_TOKEN \
                            --insecure

                        argocd app sync java-app
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
                        kubectl get svc --insecure-skip-tls-verify
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
