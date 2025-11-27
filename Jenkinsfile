pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: node
    image: node:18-alpine
    command: ['cat']
    tty: true

  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""

  - name: kubectl
    image: alpine/k8s:1.28.3
    command: ['/bin/sh', '-c']
    args: ['cat']
    tty: true
"""
    }
  }

  environment {
    IMAGE_NAME = "karankumawat/devsecops-app"
    IMAGE_TAG  = "${BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
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
        container('docker') {
          sh '''
            sleep 10
            apk add --no-cache git
            docker run --rm -v /home/jenkins/agent/workspace/devsecops-pipeline:/repo zricethezav/gitleaks:latest detect --source=/repo --no-git -v || true
          '''
        }
      }
    }

    stage('SCA - Trivy FS Scan') {
      steps {
        container('docker') {
          sh 'docker run --rm -v /home/jenkins/agent/workspace/devsecops-pipeline:/app aquasec/trivy:latest fs --scanners vuln /app || true'
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

    stage('Build & Push Image') {
      steps {
        container('docker') {
          withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
            sh '''
              cd /home/jenkins/agent/workspace/devsecops-pipeline
              docker build -t $IMAGE_NAME:$IMAGE_TAG .
              echo $PASS | docker login -u $USER --password-stdin
              docker push $IMAGE_NAME:$IMAGE_TAG
            '''
          }
        }
      }
    }

    stage('Container Image Scan') {
      steps {
        container('docker') {
          sh 'docker run --rm aquasec/trivy:latest image $IMAGE_NAME:$IMAGE_TAG || true'
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''
            cd /home/jenkins/agent/workspace/devsecops-pipeline
            kubectl apply -f k8s/deployment.yaml
            kubectl rollout status deployment/devsecops-app --timeout=2m || true
            kubectl get pods
          '''
        }
      }
    }

  }

  post {
    success {
      echo '✅ ALL STAGES PASSED!'
    }
    failure {
      echo '❌ PIPELINE FAILED'
    }
  }
}
