pipeline {
  agent any

  environment {
    BRANCH = "${env.BRANCH_NAME}"
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
          if (BRANCH == 'Dev') {
            withCredentials([file(credentialsId: 'env-dev', variable: 'ENV_FILE')]) {
              sh 'cp $ENV_FILE backend/.env'
              echo "Development environment file copied from credentials"
            }
          } else if (BRANCH == 'Main') {
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
        script {
          def branch = env.BRANCH_NAME ?: env.GIT_BRANCH
          def tag = branch
          def deployDir = "/home/jenkins/deployment"
          def envFile = (branch == "Dev") ? ".env.dev" : ".env.prod"

          echo "Deploying ${tag} environment locally on Jenkins VM..."

          try {
            sh """
            set -e
            cd ${deployDir}

            echo "Pulling latest Docker images..."
            docker pull your-docker-username/my-backend:${tag}
            docker pull your-docker-username/my-frontend:${tag}

            echo "Setting correct .env file..."
            cp ${envFile} .env

            echo "Restarting containers..."
            docker compose down
            docker compose up -d

            echo "[SUCCESS] ${tag} deployed successfully"
            """
          } catch (err) {
            error "[DEPLOY ERROR] Local deployment failed: ${err.message}"
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