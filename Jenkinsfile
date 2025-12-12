// Jenkinsfile - updated (Kubernetes pod + Kaniko + on-the-fly kubectl)
pipeline {
  agent {
    kubernetes {
      label "jenkins-node-${UUID.randomUUID().toString().take(8)}"
      defaultContainer 'node'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent-pod
spec:
  serviceAccountName: default
  containers:
  - name: node
    image: node:18-bullseye
    command:
    - cat
    tty: true
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
  - name: jnlp
    image: jenkins/inbound-agent:latest
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    tty: true
"""
    }
  }

  environment {
    IMAGE = "nexus:5000/chandarvansh/frontend"
    TAG   = "${env.BUILD_NUMBER}"
    NAMESPACE = "dev-cicd"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Use Nexus npm proxy') {
      steps {
        // ensure npm uses your nexus npm-proxy
        sh "echo 'registry=http://nexus:8081/repository/npm-proxy/' > .npmrc"
        sh "cat .npmrc"
      }
    }

    stage('Install & Build (node)') {
      steps {
        container('node') {
          sh '''
            set -e
            node --version || true
            npm --version || true
            # install dependencies and build
            npm ci
            npm run build -- --configuration production
          '''
        }
      }
    }

    stage('Prepare kubeconfig') {
      steps {
        // inject kubeconfig file from Jenkins credentials (secret file)
        withCredentials([file(credentialsId: 'jenkins-kubeconfig', variable: 'KCFG')]) {
          container('node') {
            sh '''
              mkdir -p $HOME/.kube
              cp "$KCFG" $HOME/.kube/config
              chmod 600 $HOME/.kube/config
              echo "kubectl current-context:"
              kubectl config current-context || true
            '''
          }
        }
      }
    }

    stage('Ensure kubectl installed') {
      steps {
        // install kubectl inside the node container so we don't rely on a separate image
        container('node') {
          sh '''
            set -e
            if command -v kubectl >/dev/null 2>&1; then
              echo "kubectl already present: $(kubectl version --client --short || true)"
            else
              echo "Installing kubectl..."
              STABLE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
              curl -L -o /tmp/kubectl "https://dl.k8s.io/release/${STABLE}/bin/linux/amd64/kubectl"
              chmod +x /tmp/kubectl
              sudo mv /tmp/kubectl /usr/local/bin/kubectl || (mkdir -p /usr/local/bin && mv /tmp/kubectl /usr/local/bin/kubectl)
              kubectl version --client --short || true
            fi
          '''
        }
      }
    }

    stage('Build & Push image (Kaniko)') {
      steps {
        // Use Kaniko in cluster to push to registry. It will use existing secret nexus-docker-secret in dev-cicd namespace
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          container('node') {
            sh '''
              set -e
              # tar workspace for Kaniko
              tar -czf workspace.tar.gz .

              cat > kaniko-pod.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  generateName: kaniko-build-
  namespace: ${NAMESPACE}
spec:
  restartPolicy: Never
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - "--context=tar://workspace.tar.gz"
        - "--dockerfile=/workspace/Dockerfile"
        - "--destination=${IMAGE}:${TAG}"
        - "--skip-tls-verify=true"
      volumeMounts:
        - name: workspace
          mountPath: /workspace
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: workspace
      emptyDir: {}
    - name: docker-config
      secret:
        secretName: nexus-docker-secret
YAML

              # apply pod - Kaniko will pick up workspace when we copy the tar into the pod
              kubectl -n ${NAMESPACE} apply -f kaniko-pod.yaml

              # wait for pod name
              for i in {1..20}; do
                POD=$(kubectl -n ${NAMESPACE} get pods -l job-name!=,generateName=, --no-headers -o custom-columns=":metadata.name" | grep '^kaniko-build-' || true)
                if [ -n "$POD" ]; then break; fi
                sleep 1
              done

              POD=$(kubectl -n ${NAMESPACE} get pods --no-headers -o custom-columns=":metadata.name" | grep '^kaniko-build-' || true)
              if [ -z "$POD" ]; then
                echo "kaniko pod not found; listing pods:"
                kubectl -n ${NAMESPACE} get pods -o wide
                exit 1
              fi

              echo "Kaniko pod: $POD"
              # copy context and stream logs
              kubectl -n ${NAMESPACE} cp workspace.tar.gz ${POD}:/workspace/workspace.tar.gz || true
              kubectl -n ${NAMESPACE} logs -f ${POD}

              # cleanup
              kubectl -n ${NAMESPACE} delete pod ${POD} || true
            '''
          }
        }
      }
    }

    stage('Deploy to cluster') {
      steps {
        container('node') {
          sh '''
            set -e
            # Replace image in deployment (apply fallback)
            kubectl -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record || \
              kubectl -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
          '''
        }
      }
    }
  }

  post {
    success {
      echo "Deployed ${IMAGE}:${TAG}"
    }
    failure {
      echo "Pipeline failed - see console output"
    }
  }
}
