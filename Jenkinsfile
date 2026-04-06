pipeline {
    agent any

    environment {
        IMAGE_NAME      = 'horla1/java-app:latest'
        IMAGE_REPO      = 'horla1/java-app'
        IMAGE_TAG       = 'latest'
        SONAR_HOST      = 'http://sonarqube:9000'

        // Change this if your Argo CD server is exposed on another address/port
        ARGOCD_SERVER   = 'host.docker.internal:8082'

        // Git repo Argo CD should watch
        GIT_REPO        = 'https://github.com/oaiyenitaju1-coder/java-app.git'
        GIT_BRANCH      = 'main'

        // Helm chart path inside the repo
        HELM_CHART_PATH = 'helm/java-app'
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
                    docker exec java17-builder bash -c "rm -rf /app && mkdir -p /app"
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
                    docker exec java11-tester bash -c "rm -rf /app && mkdir -p /app"
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
                        docker exec java11-tester bash -c "rm -rf /app && mkdir -p /app"
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
                          ${IMAGE_NAME}
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${IMAGE_NAME}
                    '''
                }
            }
        }

        stage('Deploy with Argo CD') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'argocd-creds',
                    usernameVariable: 'ARGOCD_USERNAME',
                    passwordVariable: 'ARGOCD_PASSWORD'
                )]) {
                    sh '''
                        docker run --rm --network host argoproj/argocd:latest \
                          argocd login ${ARGOCD_SERVER} \
                          --username "${ARGOCD_USERNAME}" \
                          --password "${ARGOCD_PASSWORD}" \
                          --insecure

                        docker run --rm --network host argoproj/argocd:latest \
                          argocd app create java-app \
                          --repo ${GIT_REPO} \
                          --path ${HELM_CHART_PATH} \
                          --revision ${GIT_BRANCH} \
                          --dest-server https://kubernetes.default.svc \
                          --dest-namespace default \
                          --sync-policy automated \
                          --upsert \
                          --helm-set image.repository=${IMAGE_REPO} \
                          --helm-set image.tag=${IMAGE_TAG} \
                          --helm-set service.nodePort=30081 \
                          --server ${ARGOCD_SERVER} \
                          --username "${ARGOCD_USERNAME}" \
                          --password "${ARGOCD_PASSWORD}" \
                          --insecure

                        docker run --rm --network host argoproj/argocd:latest \
                          argocd app sync java-app \
                          --server ${ARGOCD_SERVER} \
                          --username "${ARGOCD_USERNAME}" \
                          --password "${ARGOCD_PASSWORD}" \
                          --insecure
                    '''
                }
            }
        }

        stage('Check Pods') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        kubectl get pods -n default --insecure-skip-tls-verify
                        kubectl get svc -n default --insecure-skip-tls-verify
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
