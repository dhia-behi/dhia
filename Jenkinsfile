pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        DOCKER_IMAGE          = 'dhiaest/student-management'
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'

        K8S_DIR    = 'k8s'
        KUBECONFIG = '/home/dhia/.kube/config'
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
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} ."
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
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes (Minikube)') {
            steps {
                sh """
                    echo "=== MINIKUBE STATUS / START ==="
                    minikube status || minikube start --driver=docker

                    echo "=== K8S CONNECTIVITY ==="
                    kubectl config use-context minikube || true
                    kubectl get nodes

                    echo "=== APPLY MYSQL ==="
                    kubectl apply -f ${K8S_DIR}/mysql-secret.yaml --validate=false
                    kubectl apply -f ${K8S_DIR}/mysql-pv-pvc.yaml --validate=false
                    kubectl apply -f ${K8S_DIR}/mysql-deployment.yaml --validate=false
                    kubectl apply -f ${K8S_DIR}/mysql-service.yaml --validate=false

                    echo "=== APPLY SPRING ==="
                    kubectl apply -f ${K8S_DIR}/spring-deployment.yaml --validate=false
                    kubectl apply -f ${K8S_DIR}/spring-service.yaml --validate=false

                    echo "=== UPDATE IMAGE VERSION ==="
                    kubectl set image deployment/student-management \
                      student-management=${DOCKER_IMAGE}:${BUILD_NUMBER} || true

                    echo "=== ROLLOUT ==="
                    kubectl rollout status deployment/student-management --timeout=180s || true

                    echo "=== STATUS ==="
                    kubectl get pods -o wide
                    kubectl get svc
                """
            }
        }
    }

    post {
        success { echo "✅ SUCCESS: build + push + deploy OK" }
        failure { echo "❌ FAILURE: check logs" }
    }
}
