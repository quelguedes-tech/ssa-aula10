pipeline {
    triggers {
        // TODO: investigar e configurar a alternativa com webhook.
        // githubPush()
    }
    agent { node { label ''; customWorkspace '/workspace/devsecops-lab' } }
    options { skipDefaultCheckout(true); disableConcurrentBuilds() }
    environment { APP_IMAGE = "classroom-app:${BUILD_NUMBER}" }
    stages {
        stage('SCA - Trivy') {
            steps {
                sh '''
                    docker run --rm -v "$WORKSPACE:/src" -w /src \
                    aquasec/trivy:0.58.2 fs --scanners vuln --exit-code 0 .
                '''
            }
        }
        stage('SAST - Bandit') {
            steps {
                sh 'docker build -f ci/Dockerfile.bandit -t classroombandit:local .’
                sh '''
                    docker run --rm -v "$WORKSPACE:/src" \
                    classroom-bandit:local -r app --exit-zero
                    '''
            }
        }
        stage('Build application image') {
            steps { sh 'docker build -t "$APP_IMAGE" .' }
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
    
    post {
        always {
            sh 'docker image rm "$APP_IMAGE" 2>/dev/null || true'
        }
    }
}
