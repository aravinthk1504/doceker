pipeline {
  agent any

  environment {
    IMAGE = "static-site:latest"
    KUBE_DEPLOYMENT = "deployment.yaml"
    KUBE_SERVICE = "service.yaml"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prepare Minikube Docker Env') {
      steps {
        // Ensure Jenkins has access to minikube; using shell to eval docker env
        sh '''
        # Use minikube's docker daemon so the image is available to minikube directly
        eval $(minikube docker-env)
        docker version
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
        eval $(minikube docker-env)
        docker build -t ${IMAGE} .
        '''
      }
    }

    stage('Deploy to Minikube') {
      steps {
        sh '''
        # Apply k8s manifests (they refer to image: static-site:latest)
        kubectl apply -f ${KUBE_DEPLOYMENT}
        kubectl apply -f ${KUBE_SERVICE}
        # Force rollout to pick up new image/tag
        kubectl rollout restart deployment/static-site || true
        kubectl rollout status deployment/static-site --timeout=60s || true
        '''
      }
    }

    stage('Smoke test') {
      steps {
        sh '''
        # get NodePort
        NODE_PORT=$(kubectl get svc static-site-service -o jsonpath="{.spec.ports[0].nodePort}")
        echo "NodePort: $NODE_PORT"
        # Try curl (minikube ip)
        MINIKUBE_IP=$(minikube ip)
        echo "URL: http://$MINIKUBE_IP:$NODE_PORT"
        curl --fail --max-time 10 http://$MINIKUBE_IP:$NODE_PORT || (echo "Smoke test failed" && exit 1)
        '''
      }
    }
  }

  post {
    success {
      echo "Pipeline finished successfully."
    }
    failure {
      echo "Pipeline failed. Check logs."
    }
  }
}
