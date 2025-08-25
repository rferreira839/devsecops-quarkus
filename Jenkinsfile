pipeline {
  agent any

  environment {
    REGISTRY_HOST   = 'localhost:5112'
    IMAGE_NAME      = 'quarkus-demo'
    IMAGE_TAG       = '0.1.0'
    IMAGE_LOCAL     = "${env.REGISTRY_HOST}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
    IMAGE_CLUSTER   = "k3d-reglocal:5112/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
    TRIVY_SEVERITY  = 'HIGH,CRITICAL'
    SEMGREP_CONFIG  = 'p/ci'
  }

  options { timestamps(); skipDefaultCheckout false }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Unit Tests (Maven)') {
      steps {
        sh '''
          set -e
          # Detecta onde está o pom.xml
          if [ -f app/pom.xml ]; then SRC_DIR="app"; else SRC_DIR="."; fi
          echo "Usando SRC_DIR=$SRC_DIR"

          docker run --rm -v "$PWD/$SRC_DIR":/app -w /app maven:3.9-eclipse-temurin-17 \
            mvn -B -DskipTests=false clean package

          # Persistimos para os próximos stages
          echo "$SRC_DIR" > .srcdir
        '''
      }
      post {
        always {
          script {
            def srcdir = fileExists('.srcdir') ? readFile('.srcdir').trim() : '.'
            junit allowEmptyResults: true, testResults: "${srcdir}/**/surefire-reports/*.xml"
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -f docker/Dockerfile .
          docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_LOCAL}
        '''
      }
    }

    stage('SAST (Semgrep)') {
      steps {
        sh '''
          set -e
          SRC_DIR="."
          if [ -f .srcdir ]; then SRC_DIR=$(cat .srcdir); fi
          echo "Semgrep usando SRC_DIR=$SRC_DIR"

          docker run --rm -v "$PWD/$SRC_DIR":/src returntocorp/semgrep:latest \
            semgrep ci --config ${SEMGREP_CONFIG} --error
        '''
      }
    }

    stage('Dependency Scan (OWASP DC)') {
      steps {
        sh '''
          set -e
          SRC_DIR="."
          if [ -f .srcdir ]; then SRC_DIR=$(cat .srcdir); fi
          echo "Dependency-Check usando SRC_DIR=$SRC_DIR"

          mkdir -p reports depcache
          docker run --rm \
            -v "$PWD/$SRC_DIR":/src \
            -v "$PWD/depcache":/usr/share/dependency-check/data \
            -v "$PWD/reports":/report \
            owasp/dependency-check:latest \
            --scan /src --format "HTML" --out /report --failOnCVSS 7
        '''
      }
      post {
        always {
          publishHTML(target: [
            allowMissing: true,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: 'reports',
            reportFiles: 'dependency-check-report.html',
            reportName: 'Dependency-Check Report'
          ])
        }
      }
    }

    stage('Image Scan (Trivy)') {
      steps {
        sh '''
          set -e
          docker run --rm aquasec/trivy:latest image \
            --exit-code 1 \
            --severity ${TRIVY_SEVERITY} \
            ${IMAGE_LOCAL}
        '''
      }
    }

    stage('Push Image to Local Registry') {
      steps { sh 'docker push ${IMAGE_LOCAL}' }
    }

    stage('Deploy to DES') {
      steps {
        sh '''
          set -e
          kubectl get ns des >/dev/null 2>&1 || kubectl create ns des
          kubectl apply -k deploy/overlays/des
          kubectl -n des rollout status deploy/quarkus-demo --timeout=180s
        '''
      }
    }

    stage('Approval to Deploy PRD') {
      steps {
        script {
          timeout(time: 10, unit: 'MINUTES') {
            input message: 'Aprovar deploy em PRD?', ok: 'Aprovar'
          }
        }
      }
    }

    stage('Deploy to PRD') {
      steps {
        sh '''
          set -e
          kubectl get ns prd >/dev/null 2>&1 || kubectl create ns prd
          kubectl apply -k deploy/overlays/prd
          kubectl -n prd rollout status deploy/quarkus-demo --timeout=180s
        '''
      }
    }
  }

  post { always { archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true } }
}
