pipeline {
    agent { node { label ''; customWorkspace '/workspace/devsecops-lab' } }
    options { skipDefaultCheckout(true); disableConcurrentBuilds() }
    stages {
        stage('SCA - Trivy') {
            steps {
                sh '''
                    docker run --rm -v "$WORKSPACE:/src" -w /src \
                    aquasec/trivy:0.58.2 fs --scanners vuln --exit-code 0 .
                '''
            }
        }
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git rev-parse HEAD'
            }
        }
        stage('Verify Docker') {
            steps { sh 'docker version' }
        }
    }
}
