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
    tty: true
    volumeMounts:
    - name: docker-storage
      mountPath: /var/lib/docker

  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true
    
  volumes:
  - name: docker-storage
    emptyDir: {}
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
                      sleep 5
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

        stage('SAST - npm audit') {
            steps {
                container('node') {
                    sh 'npm audit --audit-level=moderate || true'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('dind') {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                          docker build -t $IMAGE_NAME:$IMAGE_TAG .
                          echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                          docker push $IMAGE_NAME:$IMAGE_TAG                        '''
                    }
                }
            }
        }

        stage('Container Scan - Trivy Image') {
            steps {
                container('dind') {
                    sh '''
                      docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --timeout 15m $IMAGE_NAME:$IMAGE_TAG || true
                    '''
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                container('dind') {
                    sh '''
                      APP_URL=$(kubectl get svc devsecops-app-service -o jsonpath='{.spec.clusterIP}'):3000 || echo "http://devsecops-app-service:3000"
                      docker run --rm -t zaproxy/zap-stable zap-baseline.py -t $APP_URL || true
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh '''
                      kubectl apply -f k8s/deployment.yaml
                      kubectl rollout status deployment/devsecops-app
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
            echo "Pipeline failed. Check logs above."
        }
    }
}
