pipeline {
  agent any

  options {
    disableConcurrentBuilds()
  }

  environment {
    GHCR_REGISTRY = 'ghcr.io'
    GHCR_OWNER = 'jin605'
    IMAGE_NAME = 'biddinggo-backend'
    GHCR_IMAGE_NAME = "${GHCR_REGISTRY}/${GHCR_OWNER}/${IMAGE_NAME}"

    CICD_REPO_URL = 'github.com/jin605/biddinggo-deploy.git'
    BACKEND_DEPLOYMENT_MANIFEST = 'infra/k8s/backend/deployment.yaml'
  }

  stages {
    stage('Docker Build') {
      steps {
        script {
          env.IMAGE_TAG = "${env.BUILD_NUMBER}"
        }

        sh '''
          docker build --no-cache \
            -t $GHCR_IMAGE_NAME:$IMAGE_TAG \
            -t $GHCR_IMAGE_NAME:latest \
            .
        '''

        sh 'docker image inspect $GHCR_IMAGE_NAME:$IMAGE_TAG'
      }
    }

    stage('Push to GHCR') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) {
          sh '''
            echo "$GITHUB_TOKEN" | docker login $GHCR_REGISTRY -u "$GITHUB_USER" --password-stdin
            docker push $GHCR_IMAGE_NAME:$IMAGE_TAG
            docker push $GHCR_IMAGE_NAME:latest
            docker logout $GHCR_REGISTRY
          '''
        }
      }
    }

    stage('Update Backend Deployment') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) {
          sh '''
            rm -rf cicd-repo
            git clone https://$GITHUB_USER:$GITHUB_TOKEN@$CICD_REPO_URL cicd-repo

            cd cicd-repo

            git config user.name "jenkins-bot"
            git config user.email "jenkins-bot@users.noreply.github.com"

            echo "Before:"
            grep -n "image:" $BACKEND_DEPLOYMENT_MANIFEST

            sed -i "s|image: .*|image: $GHCR_IMAGE_NAME:$IMAGE_TAG|" $BACKEND_DEPLOYMENT_MANIFEST

            echo "After:"
            grep -n "$GHCR_IMAGE_NAME:$IMAGE_TAG" $BACKEND_DEPLOYMENT_MANIFEST

            git diff -- $BACKEND_DEPLOYMENT_MANIFEST
            git add $BACKEND_DEPLOYMENT_MANIFEST
            git diff --cached --quiet && echo "No backend image change." || git commit -m "ci: update backend image to $IMAGE_TAG [skip ci]"
            git pull --rebase origin main
            git push origin main
          '''
        }
      }
    }
  }

  post {
    success {
      echo """
============================================================
  ✅ BiddingGo backend pipeline succeeded
============================================================

  🧭 Build
    Job        : ${env.JOB_NAME} #${env.BUILD_NUMBER}
    Duration   : ${currentBuild.durationString.replace(' and counting', '')}

  🐳 Image
    Version    : ${env.GHCR_IMAGE_NAME}:${env.IMAGE_TAG}

============================================================
"""
    }

    failure {
      echo """
============================================================
  ❌ BiddingGo backend pipeline failed
============================================================

  🧭 Build
    Job        : ${env.JOB_NAME} #${env.BUILD_NUMBER}
    Duration   : ${currentBuild.durationString.replace(' and counting', '')}

  🔎 Debug
    Console    : ${env.BUILD_URL}console

============================================================
"""
    }
  }
}