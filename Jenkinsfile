pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'horla1/java-app:latest'
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
                retry(3) {
                    sh '''
                        trivy image \
                          --db-repository ghcr.io/aquasecurity/trivy-db:2 \
                          --java-db-repository ghcr.io/aquasecurity/trivy-java-db:1 \
                          --cache-dir .trivycache \
                          --timeout 20m \
                          --scanners vuln \
                          --severity HIGH,CRITICAL \
                          horla1/java-app:latest
                    '''
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
                    args "--entrypoint=''"
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

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins"

                        sed -i 's|tag:.*|tag: "latest"|' gitops/values.yaml

                        git add gitops/values.yaml
                        git diff --cached --quiet && echo "No changes to commit" && exit 0

                        git commit -m "ci: update image tag to latest [skip ci]"
                        git push https://${GIT_USER}:${GIT_TOKEN}@github.com/oaiyenitaju1-coder/java-app.git main
                    '''
                }
            }
        }

        stage('Argo CD Sync') {
            agent {
                docker {
                    image 'quay.io/argoproj/argocd:v2.12.6'
                    args "--entrypoint=''"
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'argocd-creds', usernameVariable: 'ARGOCD_USER', passwordVariable: 'ARGOCD_PASS')]) {
                    sh '''
                        argocd login 192.168.49.2:8080 \
                          --username "$ARGOCD_USER" \
                          --password "$ARGOCD_PASS" \
                          --insecure

                        argocd app sync java-app --insecure

                        argocd app wait java-app \
                          --health \
                          --sync \
                          --timeout 300 \
                          --insecure

                        echo "✅ Argo CD sync complete"
                        argocd app get java-app --insecure
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
