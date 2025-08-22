pipeline {
  agent any
  options {
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout()
  }
  environment {
    REGISTRY_HOST  = 'localhost:5112'
    IMAGE_NAME     = 'quarkus-demo'
    IMAGE_TAG      = '0.1.0'
    TRIVY_SEVERITY = 'HIGH,CRITICAL'
    SEMGREP_CONFIG = 'p/ci'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Unit Tests (Maven)') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          Push-Location app
          $root = (Get-Location).Path
          docker run --rm -v "$root:/app" -w /app maven:3.9-eclipse-temurin-17 `
            mvn -B -DskipTests=false clean package
          Pop-Location
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'app/**/surefire-reports/*.xml'
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          docker build -t "$env:IMAGE_NAME:$env:IMAGE_TAG" -f docker/Dockerfile .
          docker tag "$env:IMAGE_NAME:$env:IMAGE_TAG" "$env:REGISTRY_HOST/$env:IMAGE_NAME:$env:IMAGE_TAG"
        '''
      }
    }

    stage('SAST (Semgrep)') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          $root = (Get-Location).Path
          docker run --rm -v "$root:/src" returntocorp/semgrep:latest `
            semgrep ci --config $env:SEMGREP_CONFIG --error
        '''
      }
    }

    stage('Dependency Scan (OWASP DC)') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          New-Item -ItemType Directory -Force -Path reports | Out-Null
          New-Item -ItemType Directory -Force -Path depcache | Out-Null
          $app     = (Resolve-Path .\\app).Path
          $cache   = (Resolve-Path .\\depcache).Path
          $reports = (Resolve-Path .\\reports).Path
          docker run --rm `
            -v "$app:/src" `
            -v "$cache:/usr/share/dependency-check/data" `
            -v "$reports:/report" `
            owasp/dependency-check:latest `
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
        powershell '''
          $ErrorActionPreference = "Stop"
          $image = "$env:IMAGE_NAME:$env:IMAGE_TAG"
          $work  = (Get-Location).Path
          docker save $image -o image.tar
          docker run --rm -v "$work:/work" aquasec/trivy:latest image `
            --exit-code 1 --severity $env:TRIVY_SEVERITY --input /work/image.tar
        '''
      }
    }

    stage('Push Image to Local Registry') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          $image = "$env:REGISTRY_HOST/$env:IMAGE_NAME:$env:IMAGE_TAG"
          docker push $image
        '''
      }
    }

    stage('Deploy to DES') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          kubectl create namespace des --dry-run=client -o yaml | kubectl apply -f -
          kubectl apply -k deploy/overlays/des
          kubectl -n des rollout status deploy/quarkus-demo --timeout=120s
        '''
      }
    }

    stage('Approval to Deploy PRD') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          input message: 'Aprovar deploy em PRD?', ok: 'Aprovar'
        }
      }
    }

    stage('Deploy to PRD') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          kubectl create namespace prd --dry-run=client -o yaml | kubectl apply -f -
          kubectl apply -k deploy/overlays/prd
          kubectl -n prd rollout status deploy/quarkus-demo --timeout=180s
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
    }
  }
}