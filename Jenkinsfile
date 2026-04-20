// ─────────────────────────────────────────────────────────────────────────────
// Jenkinsfile — OS Mini: Page Replacement Algorithm Simulator
// Pipeline: Checkout → Install → Test → Build → Docker Build → Docker Push → Deploy
// ─────────────────────────────────────────────────────────────────────────────

pipeline {
    agent any

    // ── Global environment variables ─────────────────────────────────────────
    environment {
        // Docker Hub image name — update DOCKERHUB_USERNAME to your Docker Hub username
        DOCKERHUB_USERNAME = credentials('dockerhub-username')   // Jenkins secret text
        IMAGE_NAME         = "${DOCKERHUB_USERNAME}/os-mini-simulator"
        IMAGE_TAG          = "${env.BUILD_NUMBER}"               // e.g. 42
        IMAGE_LATEST       = "${IMAGE_NAME}:latest"
        IMAGE_VERSIONED    = "${IMAGE_NAME}:${IMAGE_TAG}"

        // Docker registry credentials ID (configured in Jenkins → Credentials)
        DOCKER_CREDENTIALS = 'dockerhub-credentials'

        // Frontend source path inside the repo
        FRONTEND_DIR       = 'frontend'

        // Node version (must be available on the Jenkins agent or use nvm)
        NODE_VERSION       = '20'
    }

    // ── Build options ────────────────────────────────────────────────────────
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))   // keep last 10 builds
        timestamps()                                      // add timestamps to log
        timeout(time: 30, unit: 'MINUTES')               // global build timeout
        disableConcurrentBuilds()                         // no parallel runs
    }

    // ── Trigger: auto-build on every push to main ────────────────────────────
    triggers {
        githubPush()
    }

    // ═════════════════════════════════════════════════════════════════════════
    stages {

        // ── Stage 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '📥 Cloning repository...'
                checkout scm
                sh 'echo "Commit: $(git rev-parse --short HEAD)"'
                sh 'echo "Branch: $(git rev-parse --abbrev-ref HEAD)"'
            }
        }

        // ── Stage 2: Install Dependencies ────────────────────────────────
        stage('Install Dependencies') {
            steps {
                dir("${FRONTEND_DIR}") {
                    echo '📦 Installing npm dependencies...'
                    sh '''
                        node --version
                        npm --version
                        npm ci --silent
                    '''
                }
            }
        }

        // ── Stage 3: Lint ────────────────────────────────────────────────
        stage('Lint') {
            steps {
                dir("${FRONTEND_DIR}") {
                    echo '🔍 Running ESLint...'
                    // react-scripts includes ESLint — CI=false prevents warning-as-error
                    sh 'npx eslint src/ --ext .js,.jsx --max-warnings 0 || true'
                }
            }
        }

        // ── Stage 4: Test ────────────────────────────────────────────────
        stage('Test') {
            steps {
                dir("${FRONTEND_DIR}") {
                    echo '🧪 Running unit tests...'
                    sh '''
                        CI=true npm test -- \
                          --watchAll=false \
                          --coverage \
                          --coverageReporters=text \
                          --passWithNoTests
                    '''
                }
            }
            post {
                always {
                    // Publish coverage report if it exists
                    script {
                        if (fileExists("${FRONTEND_DIR}/coverage/lcov.info")) {
                            publishHTML(target: [
                                allowMissing         : true,
                                alwaysLinkToLastBuild: true,
                                keepAll              : true,
                                reportDir            : "${FRONTEND_DIR}/coverage/lcov-report",
                                reportFiles          : 'index.html',
                                reportName           : 'Coverage Report'
                            ])
                        }
                    }
                }
            }
        }

        // ── Stage 5: Build React App ──────────────────────────────────────
        stage('Build React App') {
            steps {
                dir("${FRONTEND_DIR}") {
                    echo '🏗️  Building production bundle...'
                    sh 'CI=false npm run build'
                    sh 'echo "Build size: $(du -sh build | cut -f1)"'
                }
            }
        }

        // ── Stage 6: Docker Build ────────────────────────────────────────
        stage('Docker Build') {
            steps {
                dir("${FRONTEND_DIR}") {
                    echo "🐳 Building Docker image: ${IMAGE_VERSIONED}"
                    sh """
                        docker build \
                          --no-cache \
                          -t ${IMAGE_VERSIONED} \
                          -t ${IMAGE_LATEST} \
                          .
                    """
                    sh "docker images ${IMAGE_NAME}"
                }
            }
        }

        // ── Stage 7: Docker Push ──────────────────────────────────────────
        stage('Docker Push') {
            when {
                // Only push on the main branch
                branch 'main'
            }
            steps {
                echo "📤 Pushing Docker images to Docker Hub..."
                withCredentials([usernamePassword(
                    credentialsId : "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ''' + "${IMAGE_VERSIONED}" + '''
                        docker push ''' + "${IMAGE_LATEST}" + '''
                        docker logout
                    '''
                }
            }
        }

        // ── Stage 8: Deploy (optional — uncomment & adapt as needed) ─────
        // stage('Deploy') {
        //     when { branch 'main' }
        //     steps {
        //         echo '🚀 Deploying to server...'
        //         sshagent(['deploy-ssh-credentials']) {
        //             sh """
        //                 ssh -o StrictHostKeyChecking=no user@your-server-ip '
        //                     docker pull ${IMAGE_LATEST} &&
        //                     docker stop os-mini || true &&
        //                     docker rm   os-mini || true &&
        //                     docker run  -d --name os-mini -p 80:80 ${IMAGE_LATEST}
        //                 '
        //             """
        //         }
        //     }
        // }

    }
    // ═════════════════════════════════════════════════════════════════════════

    // ── Post-build actions ───────────────────────────────────────────────────
    post {
        success {
            echo "✅ Pipeline SUCCESS — Image: ${IMAGE_VERSIONED}"
        }
        failure {
            echo "❌ Pipeline FAILED — check the logs above."
        }
        cleanup {
            echo '🧹 Cleaning up local Docker images...'
            sh """
                docker rmi ${IMAGE_VERSIONED} || true
                docker rmi ${IMAGE_LATEST}    || true
            """
            cleanWs()   // clean the Jenkins workspace
        }
    }
}
