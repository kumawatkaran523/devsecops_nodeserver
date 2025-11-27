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
    - name: docker-graph-storage
      mountPath: /var/lib/docker
  - name: kubectl
    image: bitnami/kubectl:1.28
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent
  volumes:
  - name: workspace-volume
    emptyDir: {}
  - name: docker-graph-storage
    emptyDir: {}
"""
    }
  }

  environment {
    IMAGE_NAME = "karankumawat/devsecops-app"
    IMAGE_TAG  = "v1"
  }

  options {
    timeout(time: 30, unit: 'MINUTES')
    ansiColor('xterm')
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
          sh 'npm ci'
        }
      }
    }

    stage('Secret Scan - Gitleaks') {
      steps {
        container('dind') {
          sh '''
            apk add --no-cache git
            docker run --rm -v $(pwd):/repo zricethezav/gitleaks:latest detect --source=/repo --no-git || true
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

    stage('Container Scan - Trivy Remote/Image') {
      steps {
        container('dind') {
          sh '''
            docker run --rm aquasec/trivy image --timeout 15m $IMAGE_NAME:$IMAGE_TAG || true
          '''
        }
      }
    }

    stage('DAST - OWASP ZAP (quick)') {
      steps {
        container('dind') {
          sh '''
            # run quickly against service URL inside cluster (if available)
            # fallback: skip quietly if ZAP pull/ping fails
            APP_URL="http://devsecops-app:5000"
            docker run --rm -t owasp/zap2docker-stable zap-baseline.py -t $APP_URL || true
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''
            # create/update deployment and expose (idempotent)
            kubectl set image deployment/devsecops-app devsecops-app=$IMAGE_NAME:$IMAGE_TAG --ignore-not-found || true
            kubectl get deployment devsecops-app >/dev/null 2>&1 || kubectl create deployment devsecops-app --image=$IMAGE_NAME:$IMAGE_TAG
            kubectl expose deployment devsecops-app --type=NodePort --port=5000 --name=devsecops-app-service || true
            kubectl rollout status deployment/devsecops-app --timeout=120s || true
            kubectl get pods -l app=devsecops-app -o wide || true
          '''
        }
      }
    }
  }

  post {
    success {
      echo "🎉 Pipeline finished SUCCESS"
    }
    failure {
      echo "❌ Pipeline FAILED — check console"
    }
    always {
      archiveArtifacts allowEmptyArchive: true, artifacts: '**/reports/**', onlyIfSuccessful: false
    }
  }
}
