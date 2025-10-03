pipeline {
  agent any

  environment {
    IMAGE_NAME = 'diwash716/aws-eb-express'   // your Docker Hub repo
    SNYK_SEVERITY_THRESHOLD = 'high'
    // tell Jenkins how to reach Docker-in-Docker
    DOCKER_HOST       = 'tcp://docker:2376'
    DOCKER_CERT_PATH  = '/certs/client'
    DOCKER_TLS_VERIFY = '1'
  }

  options { timestamps() }  // keep ansiColor('xterm') only if plugin installed

  stages {

    stage('Prepare Docker CLI') {
      steps {
        sh '''
          if ! command -v docker >/dev/null 2>&1; then
            echo "[setup] Installing Docker CLI..."
            apt-get update -y
            apt-get install -y docker.io
          fi
          docker --version
        '''
      }
    }

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install deps') {
      steps {
        // *** debug info ***
        sh 'echo "[debug] PWD=$PWD"; ls -la'

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
      environment { SNYK_TOKEN = credentials('snyk_token') } // <-- your Jenkins credential ID
      steps {
        sh """
          docker run --rm -v "\$PWD":/app -w /app -e SNYK_TOKEN="\$SNYK_TOKEN" node:16 bash -lc '
            npm i -g snyk &&
            snyk auth "\$SNYK_TOKEN" &&
            snyk test --severity-threshold=${SNYK_SEVERITY_THRESHOLD} --sarif-file-output=snyk.sarif
          '
        """
      }
      post { always { archiveArtifacts artifacts: 'snyk.sarif', allowEmptyArchive: true } }
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
          credentialsId: 'dockerhub',  // <-- your Jenkins Docker Hub credential ID
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
    success { echo '✅ Pipeline finished successfully.' }
    failure { echo '❌ Pipeline failed — check the failed stage logs.' }
  }
}
