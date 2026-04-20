pipeline {
    agent any

    environment {
        // 👉 Set your DockerHub username directly
        DOCKERHUB_USERNAME = "your-dockerhub-username"

        IMAGE_NAME      = "${DOCKERHUB_USERNAME}/os-mini-simulator"
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        IMAGE_LATEST    = "${IMAGE_NAME}:latest"
        IMAGE_VERSIONED = "${IMAGE_NAME}:${IMAGE_TAG}"

        // 👉 Use ONLY this credential ID in Jenkins
        DOCKER_CREDENTIALS = 'dockerhub-username'

        FRONTEND_DIR = 'frontend'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📥 Cloning repository...'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                dir("${FRONTEND_DIR}") {
                    sh '''
                        node --version
                        npm --version
                        npm install
                    '''
                }
            }
        }

        stage('Build React App') {
            steps {
                dir("${FRONTEND_DIR}") {
                    sh 'npm run build'
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir("${FRONTEND_DIR}") {
                    sh """
                        docker build -t ${IMAGE_VERSIONED} -t ${IMAGE_LATEST} .
                    """
                }
            }
        }

        stage('Docker Push') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push '"${IMAGE_VERSIONED}"'
                        docker push '"${IMAGE_LATEST}"'
                        docker logout
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ SUCCESS: ${IMAGE_VERSIONED}"
        }
        failure {
            echo "❌ FAILED"
        }
        cleanup {
            script {
                sh """
                    docker rmi ${IMAGE_VERSIONED} || true
                    docker rmi ${IMAGE_LATEST} || true
                """
            }
            cleanWs()
        }
    }
}