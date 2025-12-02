// Jenkinsfile - Declarative pipeline
pipeline {
  agent { label 'docker' } // change label to your agent that has Docker
  environment {
    IMAGE_NAME = 'hyperserve-svc'
    IMAGE_TAG  = '3'
    FULL_IMAGE = "${env.IMAGE_NAME}:${env.IMAGE_TAG}"
    CONTAINER_NAME = 'hyperserve-svc'
    HOST_PORT = '12208'
    CONTAINER_PORT = '12208'
    // Optional: set registry creds/env if you push images
  }
  options {
    // Keep build logs for 30 days, limit concurrent builds, etc.
    timestamps()
    buildDiscarder(logRotator(daysToKeepStr: '30'))
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Pre-check Docker') {
      steps {
        script {
          // Fail if Docker CLI or daemon not available
          sh '''
            echo "==> Checking docker CLI availability..."
            if ! command -v docker >/dev/null 2>&1; then
              echo "ERROR: docker CLI not found on PATH. Aborting."
              exit 2
            fi

            echo "==> Checking Docker daemon (docker info)..."
            if ! docker info >/dev/null 2>&1; then
              echo "ERROR: Docker daemon is not reachable or permission denied. Aborting."
              echo "Run this agent with Docker installed and the Jenkins user in 'docker' group or with socket access."
              exit 3
            fi

            echo "Docker is available and reachable."
            docker --version
            docker info --format '{{json .SecurityOptions}}' || true
          '''
        }
      }
    }

    stage('Build Image') {
      steps {
        script {
          // Build the docker image
          sh '''
            set -e
            echo "Building Docker image ${FULL_IMAGE}..."
            docker build -t ${FULL_IMAGE} .
            echo "Image built: ${FULL_IMAGE}"
            docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}"
          '''
        }
      }
    }

    stage('Run Container') {
      steps {
        script {
          sh '''
            set -e
            echo "Stopping any existing container named ${CONTAINER_NAME}..."
            if docker ps -a --format '{{.Names}}' | grep -w ${CONTAINER_NAME} >/dev/null 2>&1; then
              docker rm -f ${CONTAINER_NAME} || true
              echo "Removed previous container ${CONTAINER_NAME}."
            else
              echo "No existing container named ${CONTAINER_NAME}."
            fi

            echo "Starting container ${CONTAINER_NAME} mapping host ${HOST_PORT} -> container ${CONTAINER_PORT}..."
            docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:${CONTAINER_PORT} ${FULL_IMAGE}

            # Give container a couple of seconds to start
            sleep 3
            echo "Container started; container id:"
            docker ps --filter name=${CONTAINER_NAME} --format '{{.ID}}\t{{.Image}}\t{{.Ports}}'
          '''
        }
      }
    }

    stage('Verify Listening & Healthcheck') {
      steps {
        script {
          sh '''
            set -e
            echo "Collecting docker ps output..."
            docker ps -a > docker-ps.txt || true
            echo "Collecting netstat/ss output for port ${HOST_PORT}..."
            if command -v ss >/dev/null 2>&1; then
              ss -ltnp | grep ${HOST_PORT} > ss-${HOST_PORT}.txt || true
            else
              netstat -ltnp 2>/dev/null | grep ${HOST_PORT} > netstat-${HOST_PORT}.txt || true
            fi

            echo "Probing HTTP endpoint on localhost:${HOST_PORT}..."
            # try curl (if available) otherwise use wget
            if command -v curl >/dev/null 2>&1; then
              curl -sS --max-time 5 http://127.0.0.1:${HOST_PORT}/ || true > curl-${HOST_PORT}.txt
            elif command -v wget >/dev/null 2>&1; then
              wget -qO- --timeout=5 http://127.0.0.1:${HOST_PORT}/ || true > curl-${HOST_PORT}.txt
            else
              echo "curl/wget not available; skipping HTTP probe" > curl-${HOST_PORT}.txt
            fi

            ls -l docker-ps.txt ss-${HOST_PORT}.txt netstat-${HOST_PORT}.txt curl-${HOST_PORT}.txt || true
          '''
        }
      }
      post {
        success {
          archiveArtifacts artifacts: 'docker-ps.txt, ss-*.txt, netstat-*.txt, curl-*.txt', allowEmptyArchive: true
          echo "Verification artifacts archived."
        }
        always {
          // Print short summary to console
          sh 'echo "Verification files (first 50 lines)"; for f in docker-ps.txt ss-*.txt netstat-*.txt curl-*.txt; do [ -f "$f" ] && echo \"--- $f ---\" && head -n 50 \"$f\" || true; done'
        }
      }
    }
  } // stages

  post {
    failure {
      echo "Build failed. Check above logs for Pre-check Docker and other errors."
    }
    always {
      echo "Pipeline finished."
    }
  }
}
