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
                    volumeMounts:
                    - name: m2-cache
                      mountPath: /root/.m2

                  - name: docker
                    image: docker:24-cli
                    command: [sleep]
                    args: [infinity]
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock

                  - name: helm
                    image: alpine/helm:3.14.0
                    command: [sleep]
                    args: [infinity]
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
        IMAGE_NAME    = 'horla1/java-app'
        SONAR_HOST_URL = 'http://sonarqube:9000'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/oaiyenitaju1-coder/java-app.git'
            }
        }

        // ── BUILD & TEST ────────────────────────────────────────────────────────
        // Previously: docker exec java11-tester sh -lc 'mvn -B clean test'
        // Now:        runs directly inside the maven container in the agent pod
        // ────────────────────────────────────────────────────────────────────────
        stage('Build with Maven (Java 11)') {
            steps {
                container('maven') {
                    sh 'mvn -B clean test'
                }
            }
        }

        // ── SONARQUBE ───────────────────────────────────────────────────────────
        // Previously: docker exec java11-sonar sh -lc 'mvn ... sonar:sonar'
        // Now:        same maven container, sonarqube:9000 reachable via Docker
        //             network — no container name change needed
        // ────────────────────────────────────────────────────────────────────────
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

        // ── DOCKER BUILD ────────────────────────────────────────────────────────
        // Previously: sh 'docker build ...' on Jenkins controller
        // Now:        runs inside the docker container in the agent pod,
        //             which shares /var/run/docker.sock from the host
        // ────────────────────────────────────────────────────────────────────────
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

        // ── DOCKER PUSH ─────────────────────────────────────────────────────────
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

        // ── DEPLOY ──────────────────────────────────────────────────────────────
        // Unchanged from Phase 1 — helm upgrade --install via kubeconfig secret
        // Runs in the dedicated helm container (has kubectl + helm pre-installed)
        // ────────────────────────────────────────────────────────────────────────
        stage('Deploy to Kubernetes') {
            steps {
                container('helm') {
                    withCredentials([file(credentialsId: 'kubeconfig',
                                         variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG="\$KUBECONFIG_FILE"

                            echo "Current context:"
                            kubectl config current-context

                            echo "Checking cluster access..."
                            kubectl get nodes

                            echo "Deploying with Helm..."
                            helm upgrade --install java-app ./helm/java-app \
                              --set image.repository=${IMAGE_NAME} \
                              --set image.tag=${BUILD_NUMBER} \
                              --wait \
                              --timeout 2m

                            echo "Deployment status:"
                            kubectl rollout status deployment/java-app

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

