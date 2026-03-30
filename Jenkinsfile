pipeline {
    agent {
        kubernetes {
            label 'java-cicd'
            defaultContainer 'maven'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: maven
      image: maven:3.9.6-eclipse-temurin-17
      command:
        - cat
      tty: true
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock

    - name: docker
      image: docker:27.1.1-cli
      command:
        - cat
      tty: true
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock

    - name: kubectl
      image: quay.io/argoproj/argocd:v2.12.6
      command:
        - cat
      tty: true

  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
"""
        }
    }

    environment {
        APP_NAME = 'java-app'
        DOCKER_IMAGE = 'horla1/java-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
        SONAR_PROJECT_KEY = 'java-app'
        SONAR_HOST_URL = 'http://192.168.49.4:9000'
        GITOPS_FILE = 'gitops/values.yaml'

        // Change this if Jenkins cannot reach Argo CD on localhost
        ARGOCD_SERVER = 'localhost:8080'
        ARGOCD_APP = 'java-app'
    }

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                container('maven') {
                    checkout scm
                }
            }
        }

        stage('Build & Test') {
            steps {
                container('maven') {
                    sh '''
                        set -eu
                        cd /home/jenkins/agent/workspace/${JOB_NAME}
                        mvn clean test package
                    '''
                }
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            set -eu
                            cd /home/jenkins/agent/workspace/${JOB_NAME}
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:4.0.0.4121:sonar \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh '''
                        set -eu
                        cd /home/jenkins/agent/workspace/${JOB_NAME}
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} -t ${DOCKER_IMAGE}:latest .
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            set -eu
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                container('docker') {
                    sh '''
                        set -eu
                        mkdir -p /root/.cache/trivy

                        echo "Downloading Trivy vulnerability DB..."
                        n=0
                        until [ "$n" -ge 3 ]
                        do
                          trivy image --cache-dir /root/.cache/trivy --timeout 15m --download-db-only && break
                          n=$((n+1))
                          echo "Retrying Trivy DB download..."
                          sleep 10
                        done

                        echo "Running Trivy report scan (OS vulnerabilities only)..."
                        trivy image \
                          --cache-dir /root/.cache/trivy \
                          --timeout 15m \
                          --skip-db-update \
                          --skip-java-db-update \
                          --pkg-types os \
                          --exit-code 0 \
                          --severity HIGH,CRITICAL \
                          --format table \
                          --no-progress \
                          --scanners vuln \
                          ${DOCKER_IMAGE}:${DOCKER_TAG}

                        echo "Running Trivy CRITICAL gate (OS vulnerabilities only)..."
                        trivy image \
                          --cache-dir /root/.cache/trivy \
                          --timeout 15m \
                          --skip-db-update \
                          --skip-java-db-update \
                          --pkg-types os \
                          --exit-code 1 \
                          --severity CRITICAL \
                          --format json \
                          --output trivy-report.json \
                          --no-progress \
                          --scanners vuln \
                          ${DOCKER_IMAGE}:${DOCKER_TAG}
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true, allowEmptyArchive: true
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                container('maven') {
                    withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        sh '''
                            set -eu
                            cd /home/jenkins/agent/workspace/${JOB_NAME}

                            git config --global user.email "jenkins@ci.local"
                            git config --global user.name "Jenkins"
                            git config --global --add safe.directory /home/jenkins/agent/workspace/${JOB_NAME}

                            if [ ! -f "${GITOPS_FILE}" ]; then
                              echo "ERROR: ${GITOPS_FILE} not found"
                              exit 1
                            fi

                            sed -i 's|tag:.*|tag: "'"${DOCKER_TAG}"'"|' ${GITOPS_FILE}

                            git add ${GITOPS_FILE}

                            if git diff --cached --quiet; then
                              echo "No changes to commit"
                            else
                              git commit -m "Update image tag to ${DOCKER_TAG}"
                              git push https://${GIT_USER}:${GIT_TOKEN}@github.com/oaiyenitaju1-coder/java-app.git HEAD:main
                            fi
                        '''
                    }
                }
            }
        }

        stage('Verify Argo CD Sync') {
            steps {
                container('kubectl') {
                    withCredentials([string(credentialsId: 'argocd-creds', variable: 'ARGOCD_TOKEN')]) {
                        sh '''
                            set -eu

                            echo "Syncing Argo CD application..."
                            argocd app sync ${ARGOCD_APP} \
                              --server ${ARGOCD_SERVER} \
                              --insecure \
                              --auth-token $ARGOCD_TOKEN

                            echo "Waiting for Argo CD application to become healthy..."
                            argocd app wait ${ARGOCD_APP} \
                              --server ${ARGOCD_SERVER} \
                              --insecure \
                              --auth-token $ARGOCD_TOKEN \
                              --health \
                              --timeout 300
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
        always {
            cleanWs()
        }
    }
}
