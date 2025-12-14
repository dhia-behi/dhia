pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        DOCKER_IMAGE = "dhiaest/student-management"
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"
        K8S_DIR = "k8s"
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
                  kubectl apply -f ${K8S_DIR}
                  kubectl rollout status deployment/student-app
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
