pipeline {
  agent any

  environment {
    BRANCH = "${env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'dev'}"
    SSH_USER = 'TaiKhau'
    GCP_VM_DEV = '34.143.160.187'
    GCP_VM_PROD = '35.197.159.76'
    DEPLOY_DIR = "/home/TaiKhau/app"
    // Adding Docker Hub variables
    DOCKER_HUB_CREDS = credentials('dockerhub-credentials')
    DOCKER_HUB_USERNAME = "${DOCKER_HUB_CREDS_USR}"
    DOCKER_IMAGE_PREFIX = "${DOCKER_HUB_USERNAME}"
    // Adding GitHub credentials
    GITHUB_TOKEN = credentials('github-token')
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: "${BRANCH}", url: 'https://Qu11et:${GITHUB_TOKEN}@github.com/Qu11et/full-stack-fastapi-template-clone.git'
      }
    }

    stage('Build and Test') {
      steps {
        echo "Running on branch: ${BRANCH}"

        script {
          if (BRANCH == 'dev') {
            withCredentials([file(credentialsId: 'env-dev', variable: 'ENV_FILE')]) {
              sh 'cp $ENV_FILE backend/.env'
              echo "Development environment file copied from credentials"
            }
          } else if (BRANCH == 'main') {
            withCredentials([file(credentialsId: 'env-prod', variable: 'ENV_FILE')]) {
              sh 'cp $ENV_FILE backend/.env'
              echo "Production environment file copied from credentials"
            }
          }
        }

        // Run unit tests, lint, etc.
      }
    }

    stage('Build Docker Images') {
      steps {
        script {
          docker.build("${DOCKER_IMAGE_PREFIX}/my-frontend:${BRANCH}", 'frontend')
          docker.build("${DOCKER_IMAGE_PREFIX}/my-backend:${BRANCH}", 'backend')
        }
      }
    }

    stage('Push Images to Docker Hub') {
      steps {
        // Login to Docker Hub
        sh "echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_CREDS_USR} --password-stdin"
        
        // Tag images with latest tag in addition to branch tag
        sh """
          docker tag ${DOCKER_IMAGE_PREFIX}/my-backend:${BRANCH} ${DOCKER_IMAGE_PREFIX}/my-backend:latest
          docker tag ${DOCKER_IMAGE_PREFIX}/my-frontend:${BRANCH} ${DOCKER_IMAGE_PREFIX}/my-frontend:latest
          
          # Push images with branch tag
          docker push ${DOCKER_IMAGE_PREFIX}/my-backend:${BRANCH}
          docker push ${DOCKER_IMAGE_PREFIX}/my-frontend:${BRANCH}
          
          # Push images with latest tag
          docker push ${DOCKER_IMAGE_PREFIX}/my-backend:latest
          docker push ${DOCKER_IMAGE_PREFIX}/my-frontend:latest
        """
        
        // Logout from Docker Hub
        sh "docker logout"
      }
    }

    stage('Deploy') {
      steps {
        withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
          script {
            //def branch = ${BRANCH}
            def target_ip = (BRANCH == 'dev') ? GCP_VM_DEV : GCP_VM_PROD

            echo "Deploying to branch: ${BRANCH}, target VM: ${target_ip}"

            try {
              sh """
              ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@$target_ip << 'EOF'
                set -e
                trap 'echo "[ERROR] Deployment failed on \$HOSTNAME!" >&2; exit 1' ERR

                echo "Switching to deployment directory..."
                cd $DEPLOY_DIR

                echo "Pulling latest images..."
                docker pull ${DOCKER_IMAGE_PREFIX}/my-backend:${BRANCH}
                docker pull ${DOCKER_IMAGE_PREFIX}/my-frontend:${BRANCH}

                echo "Restarting services..."
                docker compose down
                docker compose up -d

                echo "[SUCCESS] Deployment finished on \$HOSTNAME"
              EOF
              """
            } catch (err) {
              error "[DEPLOY ERROR] SSH deploy to ${target_ip} failed: ${err.message}"
            }
          }
        }
      }
    }
  }

  post {
    success {
      echo "Deployment succeeded on branch ${BRANCH}"
    }
    failure {
      echo "Deployment failed on branch ${BRANCH}"
    }
  }
}