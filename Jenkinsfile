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
        DOCKER_IMAGE          = 'dhiaest/student-management'
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'

        // Kubernetes (WSL)
        K8S_DIR    = 'k8s'
        KUBECONFIG = '/home/dhiaeddinebehi/.kube/config'
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

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    set -e
                    echo "=== Docker build ==="
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                    docker images | grep -E "student-management|REPOSITORY" || true
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
                        echo "=== Docker login ==="
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin

                        echo "=== Push BUILD_NUMBER tag ==="
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        echo "=== Tag & push latest ==="
                        docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        // (Optionnel) SonarQube - active seulement si tu as configuré Sonar dans Jenkins
        // stage('MVN SONARQUBE') {
        //     steps {
        //         withSonarQubeEnv('sonar') {
        //             sh 'mvn -DskipTests sonar:sonar -Dsonar.projectKey=student-management'
        //         }
        //     }
        // }

        stage('Deploy to Kubernetes (Minikube)') {
            steps {
                sh """
                    set -e
                    echo "=== Minikube status / start ==="
                    minikube status || minikube start --driver=docker

                    echo "=== Kube context ==="
                    kubectl config use-context minikube || true
                    kubectl get nodes

                    echo "=== Apply MySQL manifests ==="
                    kubectl apply -f ${K8S_DIR}/mysql-secret.yaml --validate=false
                    kubectl apply -f ${K8S_DIR}/mysql-pv-pvc.yaml --validate=false
                    kubectl apply -f ${K8S_DIR}/mysql-deployment.yaml --validate=false
                    kubectl apply -f ${K8S_DIR}/mysql-service.yaml --validate=false

                    echo "=== Apply Spring manifests ==="
                    kubectl apply -f ${K8S_DIR}/spring-deployment.yaml --validate=false
                    kubectl apply -f ${K8S_DIR}/spring-service.yaml --validate=false

                    echo "=== Update image to ${DOCKER_IMAGE}:${BUILD_NUMBER} ==="
                    kubectl set image deployment/student-management \
                      student-management=${DOCKER_IMAGE}:${BUILD_NUMBER}

                    echo "=== Rollout status ==="
                    kubectl rollout status deployment/student-management --timeout=180s

                    echo "=== Pods / Services ==="
                    kubectl get pods -o wide
                    kubectl get svc
                """
            }
        }
    }

    post {
        success { echo "✅ SUCCESS: Build + Push DockerHub + Deploy Minikube OK" }
        failure { echo "❌ FAILURE: check stage logs" }
    }
}
