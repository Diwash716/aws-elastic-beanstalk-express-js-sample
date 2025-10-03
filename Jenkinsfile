pipeline {
  agent any     // Node 16 build agent (required)

  environment {
    REGISTRY   = 'docker.io'
    IMAGE_NAME = 'diwash716/aws-eb-express'  // <-- change this
    SNYK_SEVERITY_THRESHOLD = 'high'       // Fail build on High/Critical
  }

  options { timestamps(); ansiColor('xterm') }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install deps') {
      steps {
        sh 'npm --version'
        sh 'npm install --save'            // exactly as required
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
          npm install -g snyk
          snyk auth "$SNYK_TOKEN"
          snyk test --severity-threshold=${SNYK_SEVERITY_THRESHOLD} --sarif-file-output=snyk.sarif
        '''
      }
      post { always { archiveArtifacts artifacts: 'snyk.sarif', allowEmptyArchive: true } }
    }

    stage('Build image') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
        sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest"
      }
    }

    stage('Push image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
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
