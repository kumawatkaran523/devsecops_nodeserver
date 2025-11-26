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

  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true
"""
        }
    }

    environment {
        IMAGE_NAME = "karankumawat/devsecops-app"
        IMAGE_TAG = "v2"
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

        stage('Secret Scan - Gitleaks') {
            steps {
                container('dind') {
                    sh '''
                      apk add git
                      docker run --rm -v $(pwd):/repo zricethezav/gitleaks detect --source=/repo --no-git || true
                    '''
                }
            }
        }

        stage('SCA - Trivy FS Scan') {
            steps {
                container('dind') {
                    sh '''
                      docker run --rm -v $(pwd):/app aquasec/trivy fs --timeout 15m /app || true
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
                      docker build -t $IMAGE_NAME:$IMAGE_TAG .
                    '''
                }
            }
        }

        stage('Container Scan - Trivy Image') {
            steps {
                container('dind') {
                    sh '''
                      docker run --rm aquasec/trivy image --timeout 15m $IMAGE_NAME:$IMAGE_TAG || true
                    '''
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                container('dind') {
                    sh '''
                      docker run --rm -t owasp/zap2docker-stable zap-baseline.py -t http://localhost:5000 || true
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh '''
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
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Fix above errors."
        }
    }
}
