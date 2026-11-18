pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION   = 'us-east-1'
        ECR_REGISTRY = '131904956949.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPO     = 'cicd-demo'
        APP_HOST     = '172.31.88.169'
        APP_USER     = 'ubuntu'
        IMAGE_NAME   = 'cicd-demo'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/app" \
                      -w /app \
                      -e PYTHONPATH=/app \
                      python:3.12-slim \
                      sh -c "pip install --no-cache-dir -r requirements-dev.txt && pytest -q"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                    docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region "$AWS_REGION" \
                      | docker login --username AWS --password-stdin "$ECR_REGISTRY"

                    docker push "${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}"
                '''
            }
        }

        stage('Deploy to Application EC2') {
            steps {
                sshagent(credentials: ['app-ec2-ssh']) {
                    sh '''
                        set -eu

                        # Lab-only trust-on-first-use. Production should pre-load a verified host key.
                        ssh -o StrictHostKeyChecking=accept-new \
                            "${APP_USER}@${APP_HOST}" \
                            "bash -s -- '${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}' '${AWS_REGION}' '${ECR_REGISTRY}'" <<'REMOTE_SCRIPT'
set -euo pipefail

NEW_IMAGE="$1"
AWS_REGION="$2"
ECR_REGISTRY="$3"
CONTAINER_NAME="cicd-demo"
HEALTH_URL="http://localhost:8080/health"

# Authenticate and download the image using the Application EC2 instance role.
aws ecr get-login-password --region "$AWS_REGION" \
  | sudo docker login --username AWS --password-stdin "$ECR_REGISTRY"
sudo docker pull "$NEW_IMAGE"

# Remember the currently deployed image so we can roll back if health check fails.
OLD_IMAGE=""
if sudo docker inspect "$CONTAINER_NAME" >/dev/null 2>&1; then
    OLD_IMAGE="$(sudo docker inspect --format='{{.Config.Image}}' "$CONTAINER_NAME")"
    sudo docker rm -f "$CONTAINER_NAME"
fi

# Start the new version.
sudo docker run -d \
  --name "$CONTAINER_NAME" \
  -p 8080:8080 \
  --restart unless-stopped \
  "$NEW_IMAGE"

# Verify the application is healthy.
HEALTHY=0
for attempt in $(seq 1 12); do
    if curl --fail --silent "$HEALTH_URL"; then
        HEALTHY=1
        break
    fi
    sleep 5
done

if [ "$HEALTHY" -eq 1 ]; then
    echo "Deployment successful: $NEW_IMAGE"
    exit 0
fi

echo "Health check failed for $NEW_IMAGE; starting rollback."
sudo docker rm -f "$CONTAINER_NAME" || true

if [ -n "$OLD_IMAGE" ]; then
    sudo docker run -d \
      --name "$CONTAINER_NAME" \
      -p 8080:8080 \
      --restart unless-stopped \
      "$OLD_IMAGE"
    echo "Rollback started: $OLD_IMAGE"
else
    echo "No previous container was available to restore."
fi

exit 1
REMOTE_SCRIPT
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'CI/CD pipeline finished. Review the stage results above.'
        }
    }
}
