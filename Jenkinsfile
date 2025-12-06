pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        // Nom EXACT de ton image Docker Hub
        DOCKER_IMAGE = "dhiaest/student-management"
        // ID EXACT du credential Docker Hub dans Jenkins
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/dhia-behi/dhia.git',
                    credentialsId: 'github-dhia'
            }
        }

        stage('Build JAR (clean + package)') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // 🔥 NOUVEAU : Build de l'image Docker
        stage('Build Docker Image') {
            steps {
                sh """
                    echo "=== BUILDING DOCKER IMAGE ==="
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                """
            }
        }

        // 🔥 NOUVEAU : Push de l'image Docker vers Docker Hub
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "=== LOGIN TO DOCKER HUB ==="
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        echo "=== PUSH DOCKER IMAGE ==="
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 SUCCESS : Build + Docker Image pushed successfully!"
        }
        failure {
            echo "❌ FAILURE : Something went wrong!"
        }
    }
}
