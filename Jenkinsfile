pipeline {
  agent any
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Run script') {
      steps {
        sh 'chmod +x start_service.sh || true'
        sh './start_service.sh'
      }
    }
  }
}
