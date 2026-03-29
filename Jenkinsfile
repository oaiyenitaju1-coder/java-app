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
                        memory: "256Mi"
                        cpu: "100m"
                      limits:
                        memory: "512Mi"
                        cpu: "200m"
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                    - name: trivy-cache
                      mountPath: /root/.cache/trivy

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

                  volumes:
                  - name: m2-cache
                    emptyDir: {}
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
                  - name: trivy-cache
                    emptyDir: {}
            '''
            defaultContainer 'maven'
        }
    }

    environment {
        IMAGE_NAME      = 'horla1/java-app'
        SONAR_HOST_URL  = 'http://192.168.49.4:9000'
        GIT_REPO        = 'https://github.com/oaiyenitaju1-coder/java-app.git'
        TRIVY_CACHE_DIR = '/root/.cache/trivy'
        TRIVY_TIMEOUT   = '15m'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
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
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:4.0.0.4121:sonar \
                              -Dsonar.projectKey=java-app \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh '''
                        docker build \
                          -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                          -t ${IMAGE_NAME}:latest .
                    '''
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
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                            docker push ${IMAGE_NAME}:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                container('trivy') {
                    sh '''
                        set -eu

                        mkdir -p "${TRIVY_CACHE_DIR}"

                        echo "Downloading Trivy DBs..."
                        n=0
                        until [ "$n" -ge 3 ]
                        do
                          trivy image \
                            --cache-dir "${TRIVY_CACHE_DIR}" \
                            --timeout "${TRIVY_TIMEOUT}" \
                            --download-db-only && break
                          n=$((n+1))
                          echo "Retrying Trivy vulnerability DB download ($n/3)..."
                          sleep 10
                        done

                        n=0
                        until [ "$n" -ge 3 ]
                        do
                          trivy image \
                            --cache-dir "${TRIVY_CACHE_DIR}" \
                            --timeout "${TRIVY_TIMEOUT}" \
                            --download-java-db-only && break
                          n=$((n+1))
                          echo "Retrying Trivy Java DB download ($n/3)..."
                          sleep 10
                        done

                        echo "Running Trivy report scan..."
                        trivy image \
                          --cache-dir "${TRIVY_CACHE_DIR}" \
                          --timeout "${TRIVY_TIMEOUT}" \
                          --skip-db-update \
                          --skip-java-db-update \
                          --exit-code 0 \
                          --severity HIGH,CRITICAL \
                          --format table \
                          --no-progress \
                          --scanners vuln \
                          ${IMAGE_NAME}:${BUILD_NUMBER}

                        echo "Running Trivy CRITICAL gate..."
                        trivy image \
                          --cache-dir "${TRIVY_CACHE_DIR}" \
                          --timeout "${TRIVY_TIMEOUT}" \
                          --skip-db-update \
                          --skip-java-db-update \
                          --exit-code 1 \
                          --severity CRITICAL \
                          --format json \
                          --output trivy-report.json \
                          --no-progress \
                          --scanners vuln \
                          ${IMAGE_NAME}:${BUILD_NUMBER}
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                }
                failure {
                    echo '❌ Trivy scan failed or CRITICAL vulnerabilities were found — deploy blocked!'
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins"

                        sed -i 's|tag:.*|tag: "'"${BUILD_NUMBER}"'"|' gitops/values.yaml

                        git add gitops/values.yaml
                        git commit -m "ci: update image tag to ${BUILD_NUMBER} [skip ci]" || true
                        git push https://$GIT_USER:$GIT_TOKEN@github.com/oaiyenitaju1-coder/java-app.git main
                    '''
                }
            }
        }

        stage('Verify Argo CD Sync') {
            steps {
                container('helm') {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            export KUBECONFIG="$KUBECONFIG_FILE"

                            echo "Waiting for Argo CD to sync..."
                            sleep 30

                            echo "Deployment status:"
                            kubectl rollout status deployment/java-app --timeout=2m
                            kubectl get pods
                            kubectl get svc
                        '''
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
