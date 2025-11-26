pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:18-alpine
    command: ["cat"]
    tty: true
    securityContext:
      runAsUser: 0
  - name: dind
    image: docker:24-dind
    securityContext:
      privileged: true
    command:
    - dockerd-entrypoint.sh
    tty: true
"""
        }
    }

    environment {
        IMAGE_NAME = "karankumawat/devsecops-app"
        IMAGE_TAG = "v1"
    }

    stages {

        stage('Checkout') {
            steps {
                container('node') {
                    checkout scm
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                container('node') {
                    sh 'npm install'
                }
            }
        }

        stage('Secret Scanning - Gitleaks') {
            steps {
                container('node') {
                    sh '''
                        apk add git
                        docker run --rm -v $(pwd):/repo zricethezav/gitleaks:latest detect --source=/repo --no-git
                    '''
                }
            }
        }

        stage('SCA Scan - Trivy FS') {
            steps {
                container('dind') {
                    sh '''
                      docker run --rm -v $(pwd):/app aquasec/trivy fs /app
                    '''
                }
            }
        }

        stage('SAST - Semgrep') {
            steps {
                container('node') {
                    sh '''
                      docker run --rm -v $(pwd):/src returntocorp/semgrep semgrep scan --config auto /src || true
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('dind') {
                    sh '''
                      eval $(minikube -p minikube docker-env)
                      docker build -t $IMAGE_NAME:$IMAGE_TAG .
                    '''
                }
            }
        }

        stage('Container Scan - Trivy Image') {
            steps {
                container('dind') {
                    sh '''
                      docker run --rm aquasec/trivy image $IMAGE_NAME:$IMAGE_TAG || true
                    '''
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                container('dind') {
                    sh '''
                      docker run --rm -t owasp/zap2docker-stable zap-baseline.py -t http://devsecops-app:5000 || true
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('dind') {
                    sh '''
                      eval $(minikube -p minikube docker-env)
                      kubectl delete deploy devsecops-app --ignore-not-found
                      kubectl create deployment devsecops-app --image=$IMAGE_NAME:$IMAGE_TAG
                      kubectl expose deployment devsecops-app --type=NodePort --port=5000 || true
                    '''
                }
            }
        }

    }

    post {
        success {
            echo "DevSecOps Pipeline Completed Successfully!"
        }
        failure {
            echo "Pipeline Failed."
        }
    }
}
