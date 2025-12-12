// Jenkinsfile - full updated (kubectl fallback + safe steps)
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
    WORKDIR = "${env.WORKSPACE}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Use Nexus npm proxy') {
      steps {
        sh "echo 'registry=http://nexus:8081/repository/npm-proxy/' > .npmrc"
        sh "cat .npmrc"
      }
    }

    stage('Install & Build (node)') {
      steps {
        container('node') {
          sh '''
            set -eu
            echo "node: $(node -v) npm: $(npm -v)"
            npm ci
            npm run build -- --configuration production
          '''
        }
      }
    }

    stage('Prepare kubeconfig') {
      steps {
        withCredentials([file(credentialsId: 'jenkins-kubeconfig', variable: 'KCFG')]) {
          container('node') {
            sh '''
              set -eu
              mkdir -p "$WORKSPACE/.kube"
              cp "$KCFG" "$WORKSPACE/.kube/config"
              chmod 600 "$WORKSPACE/.kube/config"
              export KUBECONFIG="$WORKSPACE/.kube/config"
              # show context if kubectl exists (don't fail if not)
              command -v kubectl >/dev/null 2>&1 && kubectl config current-context || true
            '''
          }
        }
      }
    }

    stage('Ensure kubectl installed (no sudo)') {
      steps {
        container('node') {
          sh '''
            set -eu
            if command -v kubectl >/dev/null 2>&1; then
              echo "kubectl already present: $(kubectl version --client 2>/dev/null || true)"
            else
              echo "Installing kubectl to $HOME/.local/bin ..."
              STABLE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
              curl -L -o /tmp/kubectl "https://dl.k8s.io/release/${STABLE}/bin/linux/amd64/kubectl"
              chmod +x /tmp/kubectl
              mkdir -p "$HOME/.local/bin"
              mv /tmp/kubectl "$HOME/.local/bin/kubectl"
              export PATH="$HOME/.local/bin:$PATH"
              echo "kubectl installed: $(command -v kubectl || true)"; kubectl version --client || true
            fi
          '''
        }
      }
    }

    stage('Build & Package for Kaniko') {
      steps {
        container('node') {
          sh '''
            set -eu
            TMPCTX=$(mktemp -d)
            echo "temp ctx: $TMPCTX"
            # copy only needed files to avoid tar race
            cp -v Dockerfile "$TMPCTX/" || true
            if [ -d dist ]; then cp -r dist "$TMPCTX/"; fi
            cp -v package*.json "$TMPCTX/" || true
            ls -la "$TMPCTX"
            tar -C "$TMPCTX" -czf workspace.tar.gz .
            rm -rf "$TMPCTX"
            ls -lh workspace.tar.gz
          '''
        }
      }
    }

    stage('Build & Push image (Kaniko)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          container('node') {
            sh '''
              set -eu
              POD_NAME="kaniko-build-${BUILD_NUMBER}"

              # render pod manifest (unquoted heredoc so shell expands variables)
              cat > kaniko-pod.yaml <<YAML
apiVersion: v1
kind: Pod
metadata:
  name: ${POD_NAME}
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

              # find a usable kubectl binary (try PATH then common install dirs)
              KUBECTL=$(command -v kubectl || true)
              if [ -z "${KUBECTL}" ]; then
                if [ -x "$HOME/.local/bin/kubectl" ]; then
                  KUBECTL="$HOME/.local/bin/kubectl"
                elif [ -x "/root/.local/bin/kubectl" ]; then
                  KUBECTL="/root/.local/bin/kubectl"
                else
                  echo "kubectl not found in PATH or common locations; failing now."
                  exit 2
                fi
              fi
              echo "Using kubectl: ${KUBECTL}"

              # apply kaniko pod manifest
              ${KUBECTL} -n ${NAMESPACE} apply -f kaniko-pod.yaml

              # wait for pod to be created
              for i in {1..30}; do
                if ${KUBECTL} -n ${NAMESPACE} get pod ${POD_NAME} >/dev/null 2>&1; then break; fi
                sleep 1
              done

              # copy context (retry)
              for i in 1 2 3; do
                ${KUBECTL} -n ${NAMESPACE} cp workspace.tar.gz ${POD_NAME}:/workspace/workspace.tar.gz && break || sleep 2
              done

              # stream logs (kaniko does the build & push)
              ${KUBECTL} -n ${NAMESPACE} logs -f ${POD_NAME} || true

              # cleanup
              ${KUBECTL} -n ${NAMESPACE} delete pod ${POD_NAME} --ignore-not-found || true
            '''
          }
        }
      }
    }

    stage('Deploy to cluster') {
      steps {
        container('node') {
          sh '''
            set -eu
            # locate kubectl similarly
            KUBECTL=$(command -v kubectl || true)
            if [ -z "${KUBECTL}" ]; then
              if [ -x "$HOME/.local/bin/kubectl" ]; then
                KUBECTL="$HOME/.local/bin/kubectl"
              elif [ -x "/root/.local/bin/kubectl" ]; then
                KUBECTL="/root/.local/bin/kubectl"
              else
                echo "kubectl not found in PATH or common locations; failing now."
                exit 2
              fi
            fi
            echo "Using kubectl: ${KUBECTL}"

            export KUBECONFIG="$WORKSPACE/.kube/config"
            ${KUBECTL} -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record || ${KUBECTL} -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
          '''
        }
      }
    }
  }

  post {
    success { echo "Deployed ${IMAGE}:${TAG}" }
    failure { echo "Pipeline failed - check console output" }
  }
}
