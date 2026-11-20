pipeline {

    triggers {
        pollSCM('H/2 * * * *')
        // TODO: investigar e configurar a alternativa com webhook.
        // githubPush()
    }

    environment {
        APP_IMAGE = "classroom-app:${BUILD_NUMBER}"
        APP_CONTAINER = "classroom-app-${BUILD_NUMBER}"
        LAB_NETWORK = "classroom-net-${BUILD_NUMBER}"
    }
    agent { node { label ''; customWorkspace '/workspace/devsecops-lab' } }
    options { skipDefaultCheckout(true); disableConcurrentBuilds() }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Verify Docker') {
            steps {
                sh 'docker version'
            }
        }
        stage('Parallel Stages') {
            failFast false
            parallel {
                stage('Automated Tests') {
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
                                sh 'docker build -f ci/Dockerfile.bandit -t classroom-bandit:local .'
                                sh '''
                                    docker run --rm -v "$WORKSPACE:/src" \
                                    classroom-bandit:local -r app --exit-zero
                                '''
                            }
                        }
                        stage('Build application image') {
                            steps {
                                sh 'docker build -t "$APP_IMAGE" .'
                            }
                        }
                        stage('Start application') {
                            steps {
                                sh '''
                                    docker network create "$LAB_NETWORK"
                                    docker run -d --name "$APP_CONTAINER" --network "$LAB_NETWORK" "$APP_IMAGE"
                                    ready=0
                                    for attempt in $(seq 1 30); do
                                        if docker exec "$APP_CONTAINER" python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"; then
                                            ready=1
                                            break
                                        fi
                                        sleep 2
                                    done
                                    if [ "$ready" -ne 1 ]; then
                                        docker logs "$APP_CONTAINER"
                                        exit 1
                                    fi
                                '''
                            }
                        }
                        stage('DAST - ZAP Baseline') {
                            steps {
                                sh 'mkdir -p reports/zap; rm -f reports/zap/*; chmod 777 reports/zap'
                                script {
                                    int status = sh(returnStatus: true, script: '''
                                        docker run --rm --network "$LAB_NETWORK" \
                                        -v "$WORKSPACE/reports/zap:/zap/wrk:rw" \
                                        ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                                        -t "http://$APP_CONTAINER:8000" -m 1 \
                                        -r zap.html -J zap.json
                                    ''')
                                    if (status < 0 || status > 2) {
                                        error("ZAP execution failed with exit code ${status}")
                                    }
                                    sh 'test -s reports/zap/zap.json'
                                }
                            }
                        }
                    }
                }

                stage('WebApp for Manual Testing') {
                    steps {
                        sh '''
                            docker build -t classroom-manual:latest .
                            docker rm -f classroom-manual 2>/dev/null || true
                            docker run -d --name classroom-manual \
                                --restart unless-stopped \
                                -p 8000:8000 classroom-manual:latest

                            ready=0
                            for attempt in $(seq 1 30); do
                                if docker exec classroom-manual python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"; then
                                    ready=1
                                    break
                                fi
                                sleep 2
                            done
                            if [ "$ready" -ne 1 ]; then
                                docker logs classroom-manual
                                docker rm -f classroom-manual
                                exit 1
                            fi
                        '''
                        echo 'Aplicação disponível em http://localhost:8000 para testes manuais.'
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker logs "$APP_CONTAINER" 2>/dev/null || true'
            sh 'docker rm -f "$APP_CONTAINER" 2>/dev/null || true'
            sh 'docker network rm "$LAB_NETWORK" 2>/dev/null || true'
            sh 'docker image rm "$APP_IMAGE" 2>/dev/null || true'
            archiveArtifacts artifacts: 'reports/zap/*', allowEmptyArchive: true
        }
    }
}