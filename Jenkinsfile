pipeline {
  agent any

  environment {
    REGISTRY_HOST   = 'localhost:5112'
    IMAGE_NAME      = 'quarkus-demo'
    IMAGE_TAG       = '0.1.0' // simples pro hackathon
    IMAGE_LOCAL     = "${env.REGISTRY_HOST}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
    IMAGE_CLUSTER   = "k3d-reglocal:5112/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
    TRIVY_SEVERITY  = 'HIGH,CRITICAL'
    SEMGREP_CONFIG  = 'p/ci'
  }

  options {
  timestamps()
  skipDefaultCheckout()
  }
  stages {
    stage('Sanity') {
      steps {
        echo "Declarative OK em ${env.NODE_NAME}"
      }
    }
  }
}
