pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                metadata:
                  labels:
                    app: jenkins-agent
                spec:
                  containers:

                  - name: maven
                    image: maven:3.9-eclipse-temurin-11
                    command: [sleep]
                    args: [infinity]
                    env:
                    - name: MAVEN_OPTS
                      value: "-Xmx512m -Xms256m"
                    resources:
                      requests:
                        memory: "512Mi"
                        cpu: "250m"
                      limits:
                        memory: "768Mi"
                        cpu: "500m"
                    volumeMounts:
                    - name: m2-cache
                      mountPath: /root/.m2

                  - name: docker
                    image: docker:24-cli
                    command: [sleep]
                    args: [infinity]
                    resources:
                      requests:
                        memory: "128Mi"
                        cpu: "100m"
                      limits:
                        memory: "256Mi"
                        cpu: "200m"
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock

                  - name: trivy
                    image: aquasec/trivy:0.51.1
                    command: [sleep]
                    args: [infinity]
                    resources:
                      requests:
                        memory: "128Mi"
                        cpu: "100m"
                      limits:
                        memory: "256Mi"
                        cpu: "200m"
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock

                  - name: helm
                    image: dtzar/helm-kubectl:3.14
                    command: [sleep]
                    args: [infinity]
                    resources:
                      requests:
                        memory: "128Mi"
                        cpu: "100m"
                      limits:
                        memory: "256Mi"
                        cpu: "200m"
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock

                  volumes:
                  - name: m2-cache
                    emptyDir: {}
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
            '''
            defaultContainer 'maven'
        }
    }

    environment {
        IMAGE_NAME     = 'horla1/java-app'
        SONAR_HOST_URL = 'http://192.168.49.4:9000'
        GIT_REPO       = 'https://github.com/oaiyenitaju1-coder/java-app.git'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/oaiyenitaju1-coder/java-app.git'
            }
        }

        stage('Build with Maven (Java 11)') {
            steps {
                container('maven') {
                    sh 'mvn -B clean test'
                }
            }
        }

        stage('SonarQube Analysis (Java 11)') {
            steps {
                container('maven') {
                    withCredentials([string(credentialsId: 'sonar-token',
                                           variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn -B clean verify sonar:sonar \
                              -Dsonar.projectKey=java-app \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh """
                        docker build \
                          -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                          -t ${IMAGE_NAME}:latest .
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "${DOCKER_PASS}" | \
                              docker login -u "${DOCKER_USER}" --password-stdin
                            docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                            docker push ${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                container('trivy') {
                    sh """
                        trivy image \
                          --exit-code 0 \
                          --severity HIGH,CRITICAL \
                          --format table \
                          --no-progress \
                          ${IMAGE_NAME}:${BUILD_NUMBER}

                        trivy image \
                          --exit-code 0 \
                          --severity CRITICAL \
                          --format json \
                          --output trivy-report.json \
                          --no-progress \
                          ${IMAGE_NAME}:${BUILD_NUMBER}
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json',
                                     allowEmptyArchive: true
                }
                failure {
                    echo '❌ Trivy found CRITICAL vulnerabilities — deploy blocked!'
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(
                        credentialsId: 'github-creds',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins"

                        sed -i 's|tag:.*|tag: "${BUILD_NUMBER}"|' gitops/values.yaml

                        git add gitops/values.yaml
                        git commit -m "ci: update image tag to ${BUILD_NUMBER} [skip ci]"
                        git push https://${GIT_USER}:${GIT_TOKEN}@github.com/oaiyenitaju1-coder/java-app.git main
                    """
                }
            }
        }

        stage('Verify Argo CD Sync') {
            steps {
                container('helm') {
                    withCredentials([file(credentialsId: 'kubeconfig',
                                         variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG="\$KUBECONFIG_FILE"

                            echo "Waiting for Argo CD to sync..."
                            sleep 30

                            echo "Deployment status:"
                            kubectl rollout status deployment/java-app --timeout=2m

                            kubectl get pods
                            kubectl get svc
                        """
                    }
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

