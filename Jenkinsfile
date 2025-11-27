pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: default
  securityContext:
    runAsUser: 0

  containers:
  - name: node
    image: node:18-alpine
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent

  - name: dind
    image: docker:24-dind
    securityContext:
      privileged: true
    command:
    - dockerd-entrypoint.sh
    tty: true
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent
    - name: docker-storage
      mountPath: /var/lib/docker

  - name: kubectl
    image: lachlanevenson/k8s-kubectl:v1.28.0
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent

  volumes:
  - name: workspace-volume
    emptyDir: {}
  - name: docker-storage
    emptyDir: {}
"""
    }
  }

  environment {
    IMAGE_NAME = "karankumawat/devsecops-app"
    IMAGE_TAG  = "v3"
  }

  options {
    timeout(time: 30, unit: 'MINUTES')
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
            apk add --no-cache git
            docker run --rm -v $(pwd):/repo zricethezav/gitleaks detect --source=/repo --no-git || true
          '''
        }
      }
    }

    stage('SCA - Trivy FS Scan') {
      steps {
        container('dind') {
          sh '''
            docker run --rm -v $(pwd):/app aquasec/trivy fs --timeout 10m /app || true
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

    stage('Build & Push Image') {
      steps {
        container('dind') {
          withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh '''
              docker build -t $IMAGE_NAME:$IMAGE_TAG .
              echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
              docker push $IMAGE_NAME:$IMAGE_TAG
            '''
          }
        }
      }
    }

    stage('Container Scan - Trivy Image') {
      steps {
        container('dind') {
          sh '''
            docker run --rm aquasec/trivy image --timeout 10m $IMAGE_NAME:$IMAGE_TAG || true
          '''
        }
      }
    }

    stage('DAST - OWASP ZAP') {
      steps {
        container('dind') {
          sh '''
            TARGET_URL="http://devsecops-app:5000"
            docker run --rm -t owasp/zap2docker-stable zap-baseline.py -t $TARGET_URL || true
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''
            kubectl delete deployment devsecops-app --ignore-not-found
            kubectl create deployment devsecops-app --image=$IMAGE_NAME:$IMAGE_TAG
            kubectl expose deployment devsecops-app --type=NodePort --port=5000 --name=devsecops-service || true
            kubectl rollout status deployment/devsecops-app --timeout=120s || true
            kubectl get svc devsecops-service
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
      echo "Pipeline failed. Fix the errors above."
    }
  }
}
