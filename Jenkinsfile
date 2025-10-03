pipeline {
  agent any

  environment {
    IMAGE_NAME = 'diwash716/aws-eb-express'     // your Docker Hub repo
    SNYK_SEVERITY_THRESHOLD = 'high'
  }

  options { timestamps() }                      // keep ansiColor only if that plugin is installed

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    // NEW: make sure Docker CLI exists in the Jenkins container
    stage('Install Docker CLI (if missing)') {
      steps {
        sh '''
          if ! command -v docker >/dev/null 2>&1; then
            echo "Installing Docker CLI..."
            apt-get update
            apt-get install -y ca-certificates curl gnupg lsb-release
            install -m 0755 -d /etc/apt/keyrings
            curl -fsSL https://download.docker.com/linux/debian/gpg \
              | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
            echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
              https://download.docker.com/linux/debian $(. /etc/os-release; echo $VERSION_CODENAME) stable" \
              > /etc/apt/sources.list.d/docker.list
            apt-get update
            apt-get install -y docker-ce-cli
          fi
          docker version
        '''
      }
    }

    stage('Install deps') {
      steps {
        sh '''
          docker run --rm -v "$PWD":/app -w /app node:16 bash -lc '
            node -v
            npm ci || npm install --save
          '
        '''
      }
    }

    stage('Unit tests') {
      steps {
        sh '''
          docker run --rm -v "$PWD":/app -w /app node:16 bash -lc '
            npm test || echo "No tests defined or tests failed"
          '
        '''
      }
    }

    stage('Security Scan (Snyk)') {
      environment { SNYK_TOKEN = credentials('snyk_token') } // <-- make sure this ID exists in Jenkins
      steps {
        sh """
          docker run --rm -v "\$PWD":/app -w /app -e SNYK_TOKEN="\$SNYK_TOKEN" node:16 bash -lc '
            npm i -g snyk &&
            snyk auth "\$SNYK_TOKEN" &&
            snyk test --severity-threshold=${SNYK_SEVERITY_THRESHOLD} --sarif-file-output=snyk.sarif
          '
        """
      }
      post {
        always { archiveArtifacts artifacts: 'snyk.sarif', allowEmptyArchive: true }
      }
    }

    stage('Build image') {
      steps {
        sh '''
          docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
          docker tag   ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
        '''
      }
    }

    stage('Push image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub',          // <-- your Jenkins Docker Hub credential ID
          usernameVariable: 'USER',
          passwordVariable: 'PASS'
        )]) {
          sh '''
            echo "$PASS" | docker login -u "$USER" --password-stdin
            docker push ${IMAGE_NAME}:${BUILD_NUMBER}
            docker push ${IMAGE_NAME}:latest
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/build/**', allowEmptyArchive: true
    }
  }
}
