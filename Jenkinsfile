pipeline {
  agent any
  environment {
    IMAGE = "nexus:5000/chandarvansh/frontend"
    TAG = "${env.BUILD_NUMBER}"
    NAMESPACE = "dev-cicd"
    WORKDIR = "${env.WORKSPACE}"
  }

  stages {
    stage('Checkout') {
      steps {
        // Multibranch or Pipeline-from-SCM provides 'scm'
        checkout scm
      }
    }

    stage('Use Nexus npm proxy') {
      steps {
        sh "echo 'registry=http://nexus:8081/repository/npm-proxy/' > .npmrc"
        sh "cat .npmrc"
      }
    }

    stage('Install & Build') {
      steps {
        // run inside node container created by K8s plugin (ensure your pod template has container 'node')
        container('node') {
          sh '''
            set -e
            node --version || true
            npm --version || true
            npm ci
            npm run build -- --configuration production
          '''
        }
      }
    }

    stage('Prepare kubeconfig') {
      steps {
        // jenkins-kubeconfig should be a "Secret file" credential (stored in Jenkins)
        withCredentials([file(credentialsId: 'jenkins-kubeconfig', variable: 'KCFG')]) {
          container('node') {
            sh '''
              set -e
              mkdir -p $WORKSPACE/.kube
              cp "$KCFG" $WORKSPACE/.kube/config
              chmod 600 $WORKSPACE/.kube/config
              echo "kubectl current-context:"
              # if kubectl isn't present yet this will just print nothing; that's ok
              command -v kubectl >/dev/null 2>&1 || true
            '''
          }
        }
      }
    }

    stage('Ensure kubectl installed (no sudo)') {
      steps {
        container('node') {
          sh '''
            set -e
            # If kubectl already installed, skip
            if command -v kubectl >/dev/null 2>&1; then
              echo "kubectl present: $(command -v kubectl)"
              kubectl version --client || true
            else
              echo "Installing kubectl to $HOME/.local/bin ..."
              mkdir -p $HOME/.local/bin
              STABLE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
              curl -L -o /tmp/kubectl https://dl.k8s.io/release/${STABLE}/bin/linux/amd64/kubectl
              chmod +x /tmp/kubectl
              mv /tmp/kubectl $HOME/.local/bin/kubectl
              export PATH=$HOME/.local/bin:$PATH
              echo "kubectl installed: $(command -v kubectl)"
              kubectl version --client || true
              # ensure user PATH includes it for subsequent steps
            fi
          '''
        }
      }
    }

    stage('Build & Package for Kaniko') {
      steps {
        container('node') {
          sh '''
            set -e
            TMPCTX=$(mktemp -d)
            cp Dockerfile "$TMPCTX"/
            if [ -d dist ]; then
              cp -r dist "$TMPCTX"/
            fi
            cp -v package.json package-lock.json "$TMPCTX"/ || true
            tar -C "$TMPCTX" -czf workspace.tar.gz .
            rm -rf "$TMPCTX"
            ls -lh workspace.tar.gz || true
          '''
        }
      }
    }

    stage('Build & Push image (Kaniko)') {
      steps {
        // nexus-docker-creds is a username/password credential in Jenkins (for pushing to Nexus HTTP registry)
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          container('node') {
            sh '''
              set -euo pipefail

              POD_NAME="kaniko-build-${BUILD_NUMBER}"
              echo "Preparing kaniko pod manifest for ${POD_NAME}"

              cat > kaniko-pod.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: POD_NAME_PLACEHOLDER
  namespace: NAMESPACE_PLACEHOLDER
spec:
  restartPolicy: Never
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - "--context=tar://workspace.tar.gz"
        - "--dockerfile=/workspace/Dockerfile"
        - "--destination=IMAGE_PLACEHOLDER:TAG_PLACEHOLDER"
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

              # patch manifest placeholders
              sed -i "s|POD_NAME_PLACEHOLDER|${POD_NAME}|g" kaniko-pod.yaml
              sed -i "s|NAMESPACE_PLACEHOLDER|${NAMESPACE}|g" kaniko-pod.yaml
              sed -i "s|IMAGE_PLACEHOLDER|${IMAGE}|g" kaniko-pod.yaml
              sed -i "s|TAG_PLACEHOLDER|${TAG}|g" kaniko-pod.yaml

              # ensure kubeconfig copied earlier
              if [ ! -f "$WORKSPACE/.kube/config" ]; then
                echo "ERROR: kubeconfig missing at $WORKSPACE/.kube/config"
                exit 2
              fi
              export KUBECONFIG="$WORKSPACE/.kube/config"
              echo "Using KUBECONFIG=$KUBECONFIG"

              # ensure kubectl available (installed in previous stage)
              if command -v kubectl >/dev/null 2>&1; then
                KUBECTL_PATH=$(command -v kubectl)
              elif [ -x "$HOME/.local/bin/kubectl" ]; then
                KUBECTL_PATH="$HOME/.local/bin/kubectl"
                export PATH=$HOME/.local/bin:$PATH
              else
                echo "kubectl not found in PATH - aborting"
                exit 2
              fi
              echo "Using kubectl: ${KUBECTL_PATH}"

              # --- FIX: set insecure-skip-tls-verify on this kubeconfig's first cluster to avoid TLS verification errors ---
              # (safe for local/dev; for prod embed CA instead)
              CLUSTER_NAME=$(${KUBECTL_PATH} --kubeconfig="$KUBECONFIG" config view -o jsonpath='{.clusters[0].name}')
              echo "Setting insecure-skip-tls-verify for cluster: ${CLUSTER_NAME}"
              ${KUBECTL_PATH} --kubeconfig="$KUBECONFIG" config set-cluster "${CLUSTER_NAME}" --insecure-skip-tls-verify=true

              # apply the Kaniko pod manifest
              echo "Applying kaniko pod manifest..."
              if ! ${KUBECTL_PATH} -n ${NAMESPACE} apply -f kaniko-pod.yaml; then
                echo "kubectl apply failed. Capturing server output..."
                ${KUBECTL_PATH} -n ${NAMESPACE} get events --sort-by='.lastTimestamp' | tail -n 10 || true
                exit 3
              fi

              # wait for the pod to appear
              for i in {1..30}; do
                if ${KUBECTL_PATH} -n ${NAMESPACE} get pod ${POD_NAME} >/dev/null 2>&1; then
                  break
                fi
                sleep 1
              done

              # copy workspace (try a few times)
              for i in 1 2 3; do
                if ${KUBECTL_PATH} -n ${NAMESPACE} cp workspace.tar.gz ${POD_NAME}:/workspace/workspace.tar.gz 2>/dev/null; then
                  echo "copied workspace to ${POD_NAME}"
                  break
                fi
                echo "retrying cp..."
                sleep 2
              done

              # follow logs (kaniko does the build + push)
              echo "Streaming kaniko logs..."
              ${KUBECTL_PATH} -n ${NAMESPACE} logs -f ${POD_NAME} || true

              # delete pod
              echo "Cleaning up kaniko pod..."
              ${KUBECTL_PATH} -n ${NAMESPACE} delete pod ${POD_NAME} --ignore-not-found || true

              echo "Kaniko finished (check above logs for push result)."
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
            # Try update image on deployment; if not present, apply manifest from repo
            if kubectl -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record 2>/dev/null; then
              echo "Deployment image updated"
            else
              echo "Applying k8s manifest from repo (k8s/frontend-deployment.yaml)"
              kubectl -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
            fi
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
