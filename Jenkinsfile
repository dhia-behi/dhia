pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        // Git
        REPO_URL = 'https://github.com/dhia-behi/dhia.git'
        BRANCH   = 'main'

        // Docker Hub
        DOCKER_IMAGE          = "dhiaest/student-management"
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"

        // Kubernetes
        K8S_DIR    = "k8s"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: "${BRANCH}",
                    url: "${REPO_URL}",
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

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                  set -e

                  echo "=== KUBECONFIG: \$KUBECONFIG ==="
                  kubectl config use-context minikube || true
                  kubectl get nodes

                  echo "=== Apply manifests (no validate) ==="
                  kubectl apply -f ${K8S_DIR} --validate=false

                  echo "=== Update image ==="
                  kubectl set image deployment/student-app \
                    student-app=${DOCKER_IMAGE}:${BUILD_NUMBER}

                  echo "=== Rollout ==="
                  kubectl rollout status deployment/student-app --timeout=180s

                  echo "=== Pods / Services ==="
                  kubectl get pods -o wide
                  kubectl get svc
                """
            }
        }
    }

    post {
        success { echo '✅ Pipeline SUCCESS (Kubernetes deploy done)' }
        failure { echo '❌ Pipeline FAILED – check logs' }
    }
}
