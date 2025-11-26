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
    command:
    - cat
    tty: true
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

        stage('Build Docker Image') {
            steps {
                container('dind') {
                    sh '''
                      eval $(minikube -p minikube docker-env)
                      docker build -t devsecops-app:v1 .
                    '''
                }
            }
        }
    }
}
