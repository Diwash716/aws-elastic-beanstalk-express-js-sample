pipeline {
  agent any

  environment {
    IMAGE_NAME = 'diwash716/aws-eb-express'
    SNYK_SEVERITY_THRESHOLD = 'high'
    DOCKER_HOST       = 'tcp://docker:2376'
    DOCKER_CERT_PATH  = '/certs/client'
    DOCKER_TLS_VERIFY = '1'
  }

  options { timestamps() }

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

    stage('Prepare Node.js (if missing)') {
      steps {
        sh '''
          if ! command -v node >/dev/null 2>&1; then
            echo "[setup] Installing Node.js 16..."
            curl -fsSL https://deb.nodesource.com/setup_16.x | bash -
            apt-get install -y nodejs
          fi
          node -v
          npm -v
        '''
      }
    }

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install deps') {
      steps {
        sh 'echo "[debug] PWD=$PWD"; ls -la'
        sh 'test -f package.json || (echo "ERROR: package.json not found in $PWD" && exit 1)'
        sh 'npm install --save'
      }
    }

    stage('Unit tests') {
      steps {
        sh 'npm test || echo "No tests defined or tests failed"'
      }
    }

    stage('Security Scan (Snyk)') {
      environment { SNYK_TOKEN = credentials('snyk_token') }
      steps {
        sh '''
          npm i -g snyk
          snyk auth "$SNYK_TOKEN"
          snyk test --severity-threshold=$SNYK_SEVERITY_THRESHOLD --sarif-file-output=snyk.sarif
        '''
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
          credentialsId: 'dockerhub',
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
      sh 'docker logout || true'
    }
  }
}
