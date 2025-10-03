pipeline {
  agent any

  environment {
    IMAGE_NAME = 'diwash716/aws-eb-express'     // <-- your Docker Hub repo
    SNYK_SEVERITY_THRESHOLD = 'high'            // unused now; kept for consistency
  }

  options { timestamps() }

  stages {

    stage('Prepare Docker CLI') {
      steps {
        sh '''
          set -e
          command -v docker >/dev/null 2>&1 || {
            echo "[setup] Installing Docker CLI..."
            apt-get update -y
            apt-get install -y docker.io
          }
          docker --version
        '''
      }
    }

    stage('Prepare Node.js (if missing)') {
      steps {
        sh '''
          set -e
          if ! command -v node >/dev/null 2>&1; then
            echo "[setup] Installing Node.js 16 & npm..."
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
        sh '''
          echo "[debug] PWD=$PWD"
          ls -la
          test -f package.json
          # npm ci is preferred when lockfile present; else fallback to install
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install --save
          fi
        '''
      }
    }

    stage('Unit tests') {
      steps {
        sh '''
          set +e
          npm test
          rc=$?
          if [ $rc -ne 0 ]; then
            echo "No tests defined or tests failed (continuing per assignment)."
          fi
          exit 0
        '''
      }
    }

    stage('Security Scan (npm audit)') {
      steps {
        sh '''
          echo "[security] Running npm audit (fail on High/Critical)…"
          # --audit-level=high makes npm return non-zero if >=high vulns are present
          # --omit=dev avoids dev deps if the assignment expects prod focus
          npm audit --audit-level=high --omit=dev
        '''
      }
    }

    stage('Build image') {
      steps {
        sh '''
          echo "[debug] Using DOCKER_HOST=${DOCKER_HOST:-<not set>}"
          docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
          docker tag  ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
        '''
      }
    }

    stage('Push image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub',           // <-- your Jenkins creds ID
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${IMAGE_NAME}:${BUILD_NUMBER}
            docker push ${IMAGE_NAME}:latest
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'snyk.sarif, **/build/**', allowEmptyArchive: true
      sh 'docker logout || true'
    }
  }
}
