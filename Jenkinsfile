// Jenkinsfile — fixed kubectl installation/usage + Kaniko build
pipeline {
  agent {
    kubernetes {
      label "jnlp-node-${env.BUILD_NUMBER ?: 'manual'}"
      defaultContainer 'node'
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: default
  containers:
    - name: node
      image: node:18-bullseye
      command: ['cat']
      tty: true
      resources:
        requests:
          memory: "512Mi"
          cpu: "250m"
        limits:
          memory: "1Gi"
          cpu: "1"
"""
    }
  }

  environment {
    IMAGE = "nexus:5000/chandarvansh/frontend"
    TAG   = "${env.BUILD_NUMBER ?: 'manual'}"
    NAMESPACE = "dev-cicd"
    WORKDIR = "${env.WORKSPACE}"
    KUBECONFIG_PATH = "${env.WORKSPACE}/.kube/config"
    KUBECTL = "${env.WORKSPACE}/.local/bin/kubectl"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
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
        // jenkins-kubeconfig must be a "Secret file" credential
        withCredentials([file(credentialsId: 'jenkins-kubeconfig', variable: 'KCFG')]) {
          container('node') {
            sh '''
              set -eu
              mkdir -p "${WORKDIR}/.kube"
              cp "$KCFG" "${KUBECONFIG_PATH}"
              chmod 600 "${KUBECONFIG_PATH}"
              echo "Written kubeconfig at ${KUBECONFIG_PATH}"
              # don't call kubectl here — ensure we install it first
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
            mkdir -p "$(dirname ${KUBECTL})"
            if [ -x "${KUBECTL}" ]; then
              echo "kubectl already present: ${KUBECTL}"
            else
              echo "Installing kubectl to ${KUBECTL} ..."
              STABLE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
              curl -L -o /tmp/kubectl "https://dl.k8s.io/release/${STABLE}/bin/linux/amd64/kubectl"
              chmod +x /tmp/kubectl
              mv /tmp/kubectl "${KUBECTL}"
              echo "kubectl installed: ${KUBECTL}"
            fi
            "${KUBECTL}" --kubeconfig="${KUBECONFIG_PATH}" version --client || true
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
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          container('node') {
            sh '''
              set -eu
              export KUBECONFIG="${KUBECONFIG_PATH}"
              KCTL="${KUBECTL} --kubeconfig=${KUBECONFIG}"

              # If cert verification fails in dev, set cluster to skip tls verify (dev only)
              CLUSTER_NAME=$(${KCTL} config view -o jsonpath='{.clusters[0].name}' 2>/dev/null || true)
              if [ -n "${CLUSTER_NAME}" ]; then
                echo "Setting cluster ${CLUSTER_NAME} --insecure-skip-tls-verify=true (dev fallback)"
                ${KCTL} config set-cluster "${CLUSTER_NAME}" --insecure-skip-tls-verify=true || true
              fi

              POD_NAME="kaniko-build-${BUILD_NUMBER}"
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

              echo "Applying kaniko pod manifest..."
              if ! ${KCTL} -n ${NAMESPACE} apply -f kaniko-pod.yaml; then
                echo "kubectl apply failed — dumping events and pods:"
                ${KCTL} -n ${NAMESPACE} get events --sort-by='.lastTimestamp' | tail -n 20 || true
                ${KCTL} -n ${NAMESPACE} get pods -o wide || true
                exit 3
              fi

              # wait for pod to appear
              for i in {1..30}; do
                if ${KCTL} -n ${NAMESPACE} get pod ${POD_NAME} >/dev/null 2>&1; then break; fi
                sleep 1
              done

              # copy workspace -> try few times
              for run in 1 2 3; do
                if ${KCTL} -n ${NAMESPACE} cp workspace.tar.gz ${POD_NAME}:/workspace/workspace.tar.gz 2>/dev/null; then
                  echo "Copied workspace.tar.gz to ${POD_NAME}"
                  break
                fi
                echo "Retrying kubectl cp..."
                sleep 2
              done

              echo "Streaming kaniko logs (build+push) — this will block until complete:"
              ${KCTL} -n ${NAMESPACE} logs -f ${POD_NAME} || true

              echo "Cleaning up kaniko pod..."
              ${KCTL} -n ${NAMESPACE} delete pod ${POD_NAME} --ignore-not-found || true
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
            KCTL="${KUBECTL} --kubeconfig=${KUBECONFIG_PATH}"
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
    failure { echo "Pipeline failed — inspect console log" }
  }
}
