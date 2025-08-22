pipeline {
  agent any
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