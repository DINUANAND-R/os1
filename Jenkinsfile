pipeline {
    agent any

    environment {
        // 👉 Replace with your actual Docker Hub username
        DOCKERHUB_USERNAME = "your-dockerhub-username"

        IMAGE_NAME      = "${DOCKERHUB_USERNAME}/os-mini-simulator"
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        IMAGE_LATEST    = "${IMAGE_NAME}:latest"
        IMAGE_VERSIONED = "${IMAGE_NAME}:${IMAGE_TAG}"

        // 👉 Must match the credential ID you create in Jenkins
        DOCKER_CREDENTIALS = 'dockerhub-credentials'

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

        // ── Stage 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '📥 Cloning repository...'
                checkout scm
                sh 'git log --oneline -1'
            }
        }

        // ── Stage 2: Install Dependencies ────────────────────────────────
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

        // ── Stage 3: Build React App ──────────────────────────────────────
        stage('Build React App') {
            steps {
                dir("${FRONTEND_DIR}") {
                    // CI=false: treats warnings as warnings, not errors (avoids false failures)
                    sh 'CI=false npm run build'
                }
            }
        }

        // ── Stage 4: Docker Build ────────────────────────────────────────
        stage('Docker Build') {
            steps {
                dir("${FRONTEND_DIR}") {
                    // Double-quoted GString — variables are resolved by Groovy correctly
                    sh """
                        docker build -t ${IMAGE_VERSIONED} -t ${IMAGE_LATEST} .
                    """
                }
            }
        }

        // ── Stage 5: Docker Push ──────────────────────────────────────────
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
                    // BUG FIX: Use double-quotes so Groovy expands IMAGE_* variables.
                    // Single-quoted sh strings do NOT expand Groovy variables.
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_VERSIONED}
                        docker push ${IMAGE_LATEST}
                        docker logout
                    """
                }
            }
        }

        // ── Stage 6: Run Container (serve on port 8080) ───────────────────
        stage('Run') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    docker stop os-mini || true
                    docker rm   os-mini || true
                    docker run -d \
                        --name os-mini \
                        --restart unless-stopped \
                        -p 8080:80 \
                        ${IMAGE_LATEST}
                """
                echo "✅ App is live at http://localhost:8080"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline succeeded — image: ${IMAGE_VERSIONED}"
        }
        failure {
            echo "❌ Pipeline failed — check the stage logs above."
        }
        cleanup {
            script {
                sh """
                    docker rmi ${IMAGE_VERSIONED} || true
                    docker rmi ${IMAGE_LATEST}    || true
                """
            }
            cleanWs()
        }
    }
}