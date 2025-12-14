pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        // Docker Hub
        DOCKER_IMAGE          = "dhiaest/student-management"
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"

        // K8s manifests folder in repo
        K8S_DIR = "k8s"

        // Kubeconfig (WSL user)
        KUBECONFIG = "/home/dhiaeddinebehi/.kube/config"
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

        stage('Build Docker Image') {
            steps {
                sh """
                  set -e
                  docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKER_CREDENTIALS_ID,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                      set -e
                      echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin

                      for i in 1 2 3 4 5; do
                        echo "Docker push attempt \$i"
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER} && break
                        sleep 10
                      done

                      docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                      docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes (Minikube)') {
            steps {
                sh """
                  set -e
                  echo "=== Using KUBECONFIG: \$KUBECONFIG ==="

                  echo "=== Current context ==="
                  kubectl config current-context || true

                  echo "=== Force minikube context ==="
                  kubectl config use-context minikube || true

                  echo "=== Cluster info / nodes ==="
                  kubectl cluster-info || true
                  kubectl get nodes || true

                  echo "=== Apply manifests (disable openapi validation) ==="
                  kubectl apply -f ${K8S_DIR} --validate=false

                  echo "=== Update image to ${DOCKER_IMAGE}:${BUILD_NUMBER} (optional) ==="
                  kubectl set image deployment/student-app \
                    student-app=${DOCKER_IMAGE}:${BUILD_NUMBER} || true

                  echo "=== Rollout status ==="
                  kubectl rollout status deployment/student-app --timeout=180s

                  echo "=== Pods / Services ==="
                  kubectl get pods -o wide
                  kubectl get svc
                """
            }
        }

        stage('Smoke Test (HTTP)') {
            steps {
                sh """
                  set -e
                  echo "=== Get service URL ==="
                  APP_URL=\$(minikube service student-service --url | head -n 1)
                  echo "APP_URL=\$APP_URL"

                  echo "=== Wait + curl ==="
                  for i in {1..20}; do
                    CODE=\$(curl -s -o /dev/null -w "%{http_code}" "\$APP_URL" || true)
                    if echo "\$CODE" | grep -E "200|302|401|403" >/dev/null; then
                      echo "✅ App is responding (HTTP \$CODE)"
                      exit 0
                    fi
                    echo "Waiting app... attempt \$i (HTTP \$CODE)"
                    sleep 5
                  done

                  echo "❌ App did not respond in time"
                  exit 1
                """
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline SUCCESS'
        }
        failure {
            echo '❌ Pipeline FAILED – check logs'
        }
    }
}
