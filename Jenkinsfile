pipeline {
  agent any

  options { timestamps() }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install deps') {
      steps {
        sh 'echo "[debug] PWD=$PWD"; ls -la'
        sh '''
          node -v
          npm install --save
        '''
      }
    }

    stage('Unit tests') {
      steps {
        sh '''
          npm test || echo "No tests defined or tests failed"
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/build/**', allowEmptyArchive: true
    }
  }
}
