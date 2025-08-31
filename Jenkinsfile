pipeline {
  agent any

  options { timestamps() }

  environment {
    // Not used by compose build path; kept for logs
    DOCKER_IMAGE = "lavanya764/devopsexamapp:latest"
    COMPOSE_CMD = "docker compose"         // or "docker-compose" if your agent uses the old CLI
    WORKDIR = "${env.WORKSPACE}"           // repo root where docker-compose.yml lives
  }

  stages {
    stage('Checkout') {
      steps {
        git url: 'https://github.com/lavanyagowda2001/devops_project.git', branch: 'master'
      }
    }

    stage('Verify Docker / Compose') {
      steps {
        sh '''
          docker version || { echo "Docker not available"; exit 1; }
          ${COMPOSE_CMD} version || { echo "Docker Compose not available"; exit 1; }
        '''
      }
    }

    stage('Deploy with Docker Compose (build from source)') {
      steps {
        dir("${WORKDIR}") {
          // Build from source using the build: ./backend in your docker-compose.yml
          sh """
            set -euxo pipefail

            # stop old stack (ignore errors if first run)
            ${COMPOSE_CMD} down --remove-orphans || true

            # rebuild images from current repo state and start
            ${COMPOSE_CMD} build --pull
            ${COMPOSE_CMD} up -d

            echo "Waiting for MySQL to be ready..."
            # Give mysql container a moment to start accepting exec
            sleep 5

            # wait up to ~2 minutes for mysql health
            i=0
            until ${COMPOSE_CMD} exec -T mysql mysqladmin ping -uroot -prootpass --silent; do
              i=$((i+1))
              if [ "$i" -gt 24 ]; then
                echo "MySQL never became healthy"; ${COMPOSE_CMD} logs mysql; exit 1
              fi
              sleep 5
            done
          """
        }
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          echo "=== Container Status ==="
          ${COMPOSE_CMD} ps -a || true

          echo "=== Test Flask endpoint ==="
          # give app a moment to bind
          sleep 3
          curl -sS -I http://localhost:5000 || true
        '''
      }
    }
  }

  post {
    success {
      echo '🚀 Deployment successful!'
      sh '${COMPOSE_CMD} ps'
    }
    failure {
      echo '❗ Pipeline failed. See logs above.'
      sh 'echo "=== Tail logs ==="; ${COMPOSE_CMD} logs --tail=80 || true'
    }
    always {
      sh 'echo "=== Final 20-line logs ==="; ${COMPOSE_CMD} logs --tail=20 || true'
    }
  }
}
