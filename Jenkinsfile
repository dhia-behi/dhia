pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        // Docker Hub
        DOCKER_IMAGE = "dhiaest/student-management"
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"

        // Kubernetes
        K8S_DIR = "k8s"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"

        // SonarQube (doit matcher le nom dans Jenkins > Configure System)
        SONARQUBE_SERVER = "SonarQube"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/dhia-behi/dhia.git',
                    credentialsId: 'github-dhia'
            }
        }

        stage('Build Maven') {
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                      mvn -B -DskipTests sonar:sonar \
                        -Dsonar.projectKey=student-management \
                        -Dsonar.projectName=student-management \
                        -Dsonar.host.url=http://localhost:9000
                    '''
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  set -e
                  echo "=== Docker build ==="
                  docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                  docker images | grep -E "student-management|REPOSITORY" || true
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      set -e
                      echo "=== Docker login ==="
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                      echo "=== Push BUILD_NUMBER tag ==="
                      # retry simple en cas de coupure réseau
                      for i in 1 2 3 4 5; do
                        echo "Docker push attempt $i"
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER} && break
                        sleep 10
                        if [ "$i" -eq 5 ]; then
                          echo "❌ Docker push failed after 5 attempts"
                          exit 1
                        fi
                      done

                      echo "=== Tag & push latest ==="
                      docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                      docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                  set -e
                  echo "=== Using KUBECONFIG: $KUBECONFIG ==="

                  echo "=== Current context ==="
                  kubectl config current-context || true

                  echo "=== Force minikube context (optional) ==="
                  kubectl config use-context minikube || true

                  echo "=== Cluster info / nodes ==="
                  kubectl cluster-info || true
                  kubectl get nodes || true

                  echo "=== Apply manifests (disable openapi validation) ==="
                  kubectl apply -f ${K8S_DIR} --validate=false

                  echo "=== Update image to ${DOCKER_IMAGE}:${BUILD_NUMBER} ==="
                  kubectl set image deployment/student-app student-app=${DOCKER_IMAGE}:${BUILD_NUMBER}

                  echo "=== Rollout status ==="
                  kubectl rollout status deployment/student-app --timeout=180s

                  echo "=== Pods / Services ==="
                  kubectl get pods -o wide
                  kubectl get svc
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline SUCCESS (Maven + SonarQube + Docker + Kubernetes)'
        }
        failure {
            echo '❌ Pipeline FAILED – check logs'
        }
    }
}
