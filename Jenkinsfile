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

  options {
    timestamps()
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Use Nexus npm proxy') {
      steps {
        // ensure npm uses Nexus npm-proxy
        sh "echo 'registry=http://nexus:8081/repository/npm-proxy/' > .npmrc"
        sh "cat .npmrc"
      }
    }

    stage('Install & Build (node)') {
      steps {
        container('node') {
          sh '''
            set -euo pipefail
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
              set -euo pipefail
              mkdir -p $WORKSPACE/.kube
              cp "$KCFG" $WORKSPACE/.kube/config
              chmod 600 $WORKSPACE/.kube/config
              export KUBECONFIG=$WORKSPACE/.kube/config
              echo "kubectl context:"
              # don't fail if kubectl not present yet
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
            set -euo pipefail
            if command -v kubectl >/dev/null 2>&1; then
              echo "kubectl already present: $(kubectl version --client 2>/dev/null || true)"
            else
              echo "Installing kubectl to \$HOME/.local/bin..."
              STABLE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
              curl -L -o /tmp/kubectl "https://dl.k8s.io/release/${STABLE}/bin/linux/amd64/kubectl"
              chmod +x /tmp/kubectl
              mkdir -p $HOME/.local/bin
              mv /tmp/kubectl $HOME/.local/bin/kubectl
              export PATH=$HOME/.local/bin:$PATH
              echo "kubectl installed: $(command -v kubectl)"; kubectl version --client || true
            fi
          '''
        }
      }
    }

    stage('Build & Package for Kaniko') {
      steps {
        container('node') {
          sh '''
            set -euo pipefail
            TMPCTX=$(mktemp -d)
            echo "creating temp context: $TMPCTX"
            # copy only what Kaniko needs (Dockerfile + dist output + package*.json)
            cp -v Dockerfile "$TMPCTX/" || true
            if [ -d dist ]; then
              cp -r dist "$TMPCTX/"
            fi
            cp -v package*.json "$TMPCTX/" || true
            # verify
            ls -la "$TMPCTX"
            tar -C "$TMPCTX" -czf workspace.tar.gz .
            echo "Context tar created: $(ls -lh workspace.tar.gz)"
            rm -rf "$TMPCTX"
          '''
        }
      }
    }

    stage('Build & Push image (Kaniko)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          container('node') {
            sh '''
              set -euo pipefail
              POD_NAME="kaniko-build-${BUILD_NUMBER}"
              cat > kaniko-pod.yaml <<'YAML'
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

              # create pod
              kubectl -n ${NAMESPACE} apply -f kaniko-pod.yaml

              # wait for pod to appear
              for i in {1..30}; do
                if kubectl -n ${NAMESPACE} get pod ${POD_NAME} >/dev/null 2>&1; then break; fi
                sleep 1
              done

              # copy tar
              kubectl -n ${NAMESPACE} cp workspace.tar.gz ${POD_NAME}:/workspace/workspace.tar.gz

              # stream logs until finished
              kubectl -n ${NAMESPACE} logs -f ${POD_NAME}

              # cleanup
              kubectl -n ${NAMESPACE} delete pod ${POD_NAME} --ignore-not-found
            '''
          }
        }
      }
    }

    stage('Deploy to cluster') {
      steps {
        container('node') {
          sh '''
            set -euo pipefail
            export KUBECONFIG=$WORKSPACE/.kube/config
            # apply or update (use set image then fallback to apply)
            kubectl -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record || kubectl -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
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
      echo "Pipeline failed - check console output"
    }
  }
}
