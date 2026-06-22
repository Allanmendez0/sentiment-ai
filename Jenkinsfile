// Jenkinsfile - Pipeline CI/CD SentimentAI
pipeline {
    agent any
    
    environment {
        IMAGE_NAME = 'sentiment-ai'
        REGISTRY = 'ghcr.io/Allanmendez0'
        IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }
    
    stages {
        // Stage 1: Checkout
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branche : ${env.BRANCH_NAME}"
                echo "Commit : ${env.GIT_COMMIT}"
                sh 'git log --oneline -5'
            }
        }
        
        // Stage 2: Lint - Exécuté dans un conteneur Docker avec COPY
        stage('Lint') {
            steps {
                sh """
                    echo "Workspace : ${WORKSPACE}"
                    
                    # Créer un Dockerfile temporaire pour le lint
                    cat > Dockerfile.lint << 'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY . /app/
RUN pip install flake8 -q
CMD ["flake8", "src/", "--max-line-length=100"]
EOF

                    # Construire l'image de lint
                    docker build -f Dockerfile.lint -t lint-image .

                    # Exécuter le lint
                    docker run --rm lint-image

                    # Nettoyer
                    rm -f Dockerfile.lint
                """
            }
        }
        
        // Stage 3: Build & Test
        stage('Build & Test') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh """
                    docker run --rm \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        pytest tests/ -v \
                        --cov=src \
                        --cov-report=xml:coverage.xml \
                        --cov-report=term-missing \
                        --cov-fail-under=70
                """
            }
            post {
                failure {
                    echo 'Tests échoués ou coverage insuffisant (< 70%)'
                }
            }
        }
        
        // Stage 4: Push - Uniquement sur la branche main
        stage('Push') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASS'
                )]) {
                    sh """
                        echo \$REGISTRY_PASS | docker login ghcr.io -u \$REGISTRY_USER --password-stdin
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest
                        docker push ${REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker compose down -v 2>/dev/null || true'
        }
        success {
            echo "Pipeline réussi ! Image : ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo 'Pipeline échoué. Consultez les logs ci-dessus.'
        }
    }
}