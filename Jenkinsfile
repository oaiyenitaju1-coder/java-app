pipeline {
    agent {
        kubernetes {
            defaultContainer 'maven'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: java-cicd
spec:
  serviceAccountName: jenkins
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
      image: docker:27.0.3-cli
      command:
        - cat
      tty: true
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock

    - name: trivy
      image: aquasec/trivy:0.57.1
      command:
        - sh
        - -c
        - cat
      tty: true
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock

    - name: argocd
      image: quay.io/argoproj/argocd:v2.12.6
      command:
        - sh
        - -c
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
        APP_NAME          = 'java-app'
        IMAGE_REPO        = 'horla1/java-app'
        IMAGE_TAG         = "${BUILD_NUMBER}"
        SONAR_HOST_URL    = 'http://192.168.49.4:9000'
        SONAR_PROJECT_KEY = 'java-app'
        ARGOCD_SERVER     = '192.168.49.4:8080'
        ARGOCD_APP        = 'java-app'
        WORK_DIR          = '/home/jenkins/agent/workspace/ola-cicd'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                    set -eu
                    pwd
                    ls -la
                '''
            }
        }

        stage('Build & Unit Test') {
            steps {
                container('maven') {
                    sh '''
                        set -eu
                        cd "$WORK_DIR"
                        mvn clean test package
                    '''
                }
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                    archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            set -eu
                            cd "$WORK_DIR"
                            mvn sonar:sonar \
                              -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                              -Dsonar.host.url="$SONAR_HOST_URL" \
                              -Dsonar.login="$SONAR_TOKEN"
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
                        cd "$WORK_DIR"
                        docker build -t "$IMAGE_REPO:$IMAGE_TAG" -t "$IMAGE_REPO:latest" .
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            set -eu
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push "$IMAGE_REPO:$IMAGE_TAG"
                            docker push "$IMAGE_REPO:latest"
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
                        mkdir -p /root/.cache/trivy

                        echo 'Downloading Trivy vulnerability DB...'
                        n=0
                        until [ "$n" -ge 3 ]
                        do
                          trivy image --cache-dir /root/.cache/trivy --timeout 15m --download-db-only && break
                          n=$((n+1))
                          echo "Retrying Trivy DB download... attempt $((n+1))"
                          sleep 10
                        done

                        echo 'Running Trivy report scan (OS vulnerabilities only)...'
                        trivy image \
                          --cache-dir /root/.cache/trivy \
                          --timeout 15m \
                          --skip-db-update \
                          --skip-java-db-update \
                          --vuln-type os \
                          --exit-code 0 \
                          --severity HIGH,CRITICAL \
                          --format table \
                          --no-progress \
                          --scanners vuln \
                          "$IMAGE_REPO:$IMAGE_TAG"

                        echo 'Running Trivy CRITICAL gate (OS vulnerabilities only)...'
                        trivy image \
                          --cache-dir /root/.cache/trivy \
                          --timeout 15m \
                          --skip-db-update \
                          --skip-java-db-update \
                          --vuln-type os \
                          --exit-code 1 \
                          --severity CRITICAL \
                          --format json \
                          --output trivy-report.json \
                          --no-progress \
                          --scanners vuln \
                          "$IMAGE_REPO:$IMAGE_TAG"
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                }
                failure {
                    echo '❌ Trivy scan failed or CRITICAL OS vulnerabilities were found — deploy blocked!'
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                container('maven') {
                    withCredentials([string(credentialsId: 'git-token', variable: 'GIT_TOKEN')]) {
                        sh '''
                            set -eu
                            cd "$WORK_DIR"

                            git config --global user.email "jenkins@ci.local"
                            git config --global user.name "Jenkins"
                            git config --global --add safe.directory "$WORK_DIR"

                            if [ ! -f gitops/values.yaml ]; then
                              echo "gitops/values.yaml not found"
                              exit 1
                            fi

                            sed -i 's|tag:.*|tag: "'"$IMAGE_TAG"'"|' gitops/values.yaml

                            git add gitops/values.yaml

                            if git diff --cached --quiet; then
                              echo "No GitOps changes to commit"
                            else
                              git commit -m "Update image tag to $IMAGE_TAG"
                              git push "https://horla1:$GIT_TOKEN@github.com/horla1/ola-cicd.git" HEAD:main
                            fi
                        '''
                    }
                }
            }
        }

        stage('Verify Argo CD Sync') {
            steps {
                container('argocd') {
                    withCredentials([usernamePassword(credentialsId: 'argocd-creds', usernameVariable: 'ARGOCD_USER', passwordVariable: 'ARGOCD_PASS')]) {
                        sh '''
                            set -eu
                            argocd login "$ARGOCD_SERVER" \
                              --username "$ARGOCD_USER" \
                              --password "$ARGOCD_PASS" \
                              --insecure

                            argocd app sync "$ARGOCD_APP" --insecure
                            argocd app wait "$ARGOCD_APP" --health --sync --timeout 300 --insecure
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully. Image: ${IMAGE_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        always {
            script {
                if (env.WORKSPACE?.trim()) {
                    cleanWs(deleteDirs: true, notFailBuild: true)
                }
            }
        }
    }
}
