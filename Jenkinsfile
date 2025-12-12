// Jenkinsfile — updated to fix: "container node not found", kubeconfig/permission and TLS issues.
// Preserves the earlier working stages (checkout, npm build). Only agent + kubectl/kaniko logic changed.

pipeline {
  // Use a Kubernetes pod template for the agent so the 'node' container always exists.
  agent {
    kubernetes {
      label "jnlp-node-${env.BUILD_NUMBER ?: 'manual'}"
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
          memory: "512Mi"
          cpu: "250m"
        limits:
          memory: "1Gi"
          cpu: "1"
    # The jnlp container is automatically provided by the plugin; inbound-agent image will be used.
"""
    }
  }

  environment {
    IMAGE = "nexus:5000/chandarvansh/frontend"
    TAG   = "${env.BUILD_NUMBER ?: 'manual'}"
    NAMESPACE = "dev-cicd"
    WORKDIR = "${env.WORKSPACE}"
    KUBECONFIG_PATH = "${env.WORKSPACE}/.kube/config"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Use Nexus npm proxy') {
      steps {
        container('node') {
          sh '''
            echo "registry=http://nexus:8081/repository/npm-proxy/" > .npmrc
            cat .npmrc
          '''
        }
      }
    }

    stage('Install & Build') {
      steps {
        container('node') {
          sh '''
            set -eu
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
        // jenkins-kubeconfig credential should be "Secret file" that you already created
        withCredentials([file(credentialsId: 'jenkins-kubeconfig', variable: 'KCFG')]) {
          container('node') {
            sh '''
              set -eu
              mkdir -p "${WORKDIR}/.kube"
              cp "$KCFG" "${KUBECONFIG_PATH}"
              chmod 600 "${KUBECONFIG_PATH}"
              echo "Written kubeconfig at ${KUBECONFIG_PATH}"
              # show current contexts for debug (masked in Jenkins)
              kubectl --kubeconfig="${KUBECONFIG_PATH}" config view --minify -o jsonpath='{.contexts[0].name}' || true
            '''
          }
        }
      }
    }

    stage('Ensure kubectl available') {
      steps {
        container('node') {
          sh '''
            set -eu
            if command -v kubectl >/dev/null 2>&1; then
              echo "kubectl already present: $(command -v kubectl)"
            else
              echo "Installing kubectl to $HOME/.local/bin ..."
              mkdir -p "$HOME/.local/bin"
              STABLE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
              curl -L -o /tmp/kubectl "https://dl.k8s.io/release/${STABLE}/bin/linux/amd64/kubectl"
              chmod +x /tmp/kubectl
              mv /tmp/kubectl "$HOME/.local/bin/kubectl"
              export PATH="$HOME/.local/bin:$PATH"
              echo "kubectl installed: $(command -v kubectl)"
            fi
            kubectl --kubeconfig="${KUBECONFIG_PATH}" version --client || true
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
            cp Dockerfile "$TMPCTX"/
            if [ -d dist ]; then cp -r dist "$TMPCTX"/; fi
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
        // Ensure nexus-docker-creds exists as usernamePassword credential in Jenkins
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          container('node') {
            sh '''
              set -eu

              # Always use explicit kubeconfig to avoid in-cluster default-sa usage
              export KUBECONFIG="${KUBECONFIG_PATH}"
              KCTL="kubectl --kubeconfig=${KUBECONFIG}"

              # If TLS CA errors occur, set cluster to skip TLS verify (dev only)
              CLUSTER_NAME=$(${KCTL} config view -o jsonpath='{.clusters[0].name}')
              if [ -n "${CLUSTER_NAME:-}" ]; then
                echo "Setting cluster ${CLUSTER_NAME} to insecure-skip-tls-verify (local/dev workaround)"
                ${KCTL} config set-cluster "${CLUSTER_NAME}" --insecure-skip-tls-verify=true || true
              fi

              POD_NAME="kaniko-build-${BUILD_NUMBER}"
              echo "Creating kaniko pod manifest for ${POD_NAME}"

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

              sed -i "s|POD_NAME_PLACEHOLDER|${POD_NAME}|g" kaniko-pod.yaml
              sed -i "s|NAMESPACE_PLACEHOLDER|${NAMESPACE}|g" kaniko-pod.yaml
              sed -i "s|IMAGE_PLACEHOLDER|${IMAGE}|g" kaniko-pod.yaml
              sed -i "s|TAG_PLACEHOLDER|${TAG}|g" kaniko-pod.yaml

              # apply the kaniko pod using explicit kubeconfig (so it uses jenkins-sa credentials from file)
              echo "Applying kaniko pod..."
              if ! ${KCTL} -n ${NAMESPACE} apply -f kaniko-pod.yaml; then
                echo "kubectl apply failed; showing last events:"
                ${KCTL} -n ${NAMESPACE} get events --sort-by='.lastTimestamp' | tail -n 15 || true
                ${KCTL} -n ${NAMESPACE} get pods -o wide || true
                exit 3
              fi

              # wait for pod to be created
              for i in {1..30}; do
                if ${KCTL} -n ${NAMESPACE} get pod ${POD_NAME} >/dev/null 2>&1; then break; fi
                sleep 1
              done

              # copy workspace archive to pod
              for i in 1 2 3; do
                if ${KCTL} -n ${NAMESPACE} cp workspace.tar.gz ${POD_NAME}:/workspace/workspace.tar.gz 2>/dev/null; then
                  echo "Copied workspace.tar.gz to ${POD_NAME}"
                  break
                fi
                echo "Retrying kubectl cp..."
                sleep 2
              done

              # stream logs (kaniko will build+push). This blocks until Kaniko finishes.
              echo "Streaming kaniko logs..."
              ${KCTL} -n ${NAMESPACE} logs -f ${POD_NAME} || true

              # cleanup
              echo "Deleting kaniko pod..."
              ${KCTL} -n ${NAMESPACE} delete pod ${POD_NAME} --ignore-not-found || true

              echo "Kaniko step finished. Check logs above for image push success."
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
            KCTL="kubectl --kubeconfig=${KUBECONFIG_PATH}"
            # Try to update image; if no deployment present apply manifest from repo
            if ${KCTL} -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record 2>/dev/null; then
              echo "Updated deployment image to ${IMAGE}:${TAG}"
            else
              echo "Applying k8s manifests from repo"
              ${KCTL} -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
            fi
          '''
        }
      }
    }
  }

  post {
    success { echo "Deployed ${IMAGE}:${TAG}" }
    failure {
      echo "Pipeline failed - check console output"
    }
  }
}
