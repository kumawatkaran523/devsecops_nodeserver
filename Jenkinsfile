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
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run

  - name: kubectl
    image: bitnami/kubectl:1.28
    command: ['cat']
    tty: true

  volumes:
  - name: docker-sock
    emptyDir: {}
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
            sleep 5
            apk add --no-cache git
            docker run --rm -v $PWD:/repo zricethezav/gitleaks:latest detect --source=/repo --no-git -v || true
          '''
        }
      }
    }

    stage('SCA - Trivy FS Scan') {
      steps {
        container('docker') {
          sh 'docker run --rm -v $PWD:/app aquasec/trivy:latest fs --scanners vuln /app || true'
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
          sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image $IMAGE_NAME:$IMAGE_TAG || true'
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''
            kubectl apply -f k8s/deployment.yaml
            kubectl set image deployment/devsecops-app devsecops-app=$IMAGE_NAME:$IMAGE_TAG
            kubectl rollout status deployment/devsecops-app --timeout=2m
          '''
        }
      }
    }

  }

  post {
    success {
      echo '✅ DevSecOps Pipeline SUCCESS!'
    }
    failure {
      echo '❌ Pipeline FAILED'
    }
  }
}
