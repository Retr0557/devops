pipeline {
  agent any

  environment {
    IMAGE_NAME = "hyperserve-svc"
    IMAGE_TAG  = "3"
    IMAGE_FULL = "${env.IMAGE_NAME}:${env.IMAGE_TAG}"
    CONTAINER_NAME = "hyperserve-svc-3"
    HOST_PORT = "12208"
    CONTAINER_PORT = "12208"
  }

  stages {
    stage('Pre-check Docker') {
      steps {
        script {
          echo "Checking Docker availability..."
          // fail the pipeline with clear message if docker isn't available or returns non-zero
          def rc = sh(script: 'docker version --format "{{.Server.Version}}" > /dev/null 2>&1 || echo $? ', returnStdout: true).trim()
          if (rc != "") {
            // If rc contains a number it may be from echo $? fallback; detect docker failure by trying docker ps
            def ok = sh(script: 'docker ps > /dev/null 2>&1; echo $?', returnStdout: true).trim()
            if (ok != '0') {
              error("""Docker does not seem to be available on the agent.
Check that Docker is installed and the Jenkins agent user has permission to run Docker.
Command 'docker ps' failed with exit code ${ok}.""")
            }
          }
          echo "Docker is available."
        }
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build image') {
      steps {
        echo "Building Docker image ${IMAGE_FULL}..."
        sh """
          # remove intermediate image with same tag if exists (optional)
          docker build -t ${IMAGE_FULL} .
        """
      }
    }

    stage('Stop previous container (if any)') {
      steps {
        echo "Stopping and removing any existing container named ${CONTAINER_NAME}..."
        sh """
          if docker ps -a --format '{{.Names}}' | grep -x ${CONTAINER_NAME} > /dev/null 2>&1; then
            docker rm -f ${CONTAINER_NAME} || true
          fi
        """
      }
    }

    stage('Run container') {
      steps {
        echo "Starting container ${CONTAINER_NAME} mapping host ${HOST_PORT} -> container ${CONTAINER_PORT}..."
        sh """
          docker run -d --rm --name ${CONTAINER_NAME} -p ${HOST_PORT}:${CONTAINER_PORT} ${IMAGE_FULL}
        """
      }
    }

    stage('Smoke test') {
      steps {
        echo "Waiting for service to respond on http://localhost:${HOST_PORT} ..."
        // small loop to try curl a few times
        sh '''
          n=0
          until [ $n -ge 10 ]
          do
            if curl -fsS --max-time 2 http://localhost:${HOST_PORT}/ >/dev/null 2>&1; then
              echo "Service responded"
              exit 0
            fi
            n=$((n+1))
            sleep 1
          done
          echo "Service did not respond within timeout"
          exit 2
        '''
      }
    }
  }

  post {
    success {
      echo "Pipeline completed successfully. image: ${IMAGE_FULL}, container: ${CONTAINER_NAME} listening on ${HOST_PORT}."
    }
    failure {
      echo "Pipeline failed — please check the stage logs above."
    }
  }
}
