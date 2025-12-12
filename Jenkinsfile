pipeline {
  agent any
  environment {
    IMAGE = "nexus:5000/chandarvansh/frontend"
    TAG = "${env.BUILD_NUMBER}"
    NAMESPACE = "dev-cicd"
  }

  stages {
    stage('Checkout SCM') {
      steps { checkout scm }
    }

    stage('Use Nexus npm proxy') {
      steps {
        sh "echo 'registry=http://nexus:8081/repository/npm-proxy/' > .npmrc"
        sh "cat .npmrc"
      }
    }

    stage('Install & Build') {
      steps {
        container('node') {
          sh 'npm ci'
          sh 'npm run build -- --configuration production'
        }
      }
    }

    stage('Prepare kubeconfig') {
      steps {
        withCredentials([file(credentialsId: 'jenkins-kubeconfig', variable: 'KCFG')]) {
          sh '''
            mkdir -p $WORKSPACE/.kube
            cp "$KCFG" $WORKSPACE/.kube/config
            chmod 600 $WORKSPACE/.kube/config
            export KUBECONFIG=$WORKSPACE/.kube/config
            kubectl config current-context || true
          '''
        }
      }
    }

    stage('Ensure kubectl available') {
      steps {
        container('node') {
          sh '''
            set -e
            mkdir -p $WORKSPACE/.local/bin
            STABLE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
            curl -L -o /tmp/kubectl "https://dl.k8s.io/release/${STABLE}/bin/linux/amd64/kubectl"
            chmod +x /tmp/kubectl
            mv /tmp/kubectl $WORKSPACE/.local/bin/kubectl
            export PATH=$WORKSPACE/.local/bin:$PATH
            kubectl --kubeconfig=$WORKSPACE/.kube/config version --client || true
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
            cp Dockerfile "${TMPCTX}/" || true
            [ -d dist ] && cp -r dist "${TMPCTX}/"
            cp -v package.json package-lock.json "${TMPCTX}/" || true
            tar -C "${TMPCTX}" -czf workspace.tar.gz .
            rm -rf "${TMPCTX}"
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

              export KUBECONFIG=$WORKSPACE/.kube/config
              KUBECTL="$WORKSPACE/.local/bin/kubectl --kubeconfig=$KUBECONFIG"

              # If cluster name present, avoid strict validation for dev clusters (keeps earlier behavior)
              CLUSTER_NAME=$($KUBECTL config view -o jsonpath='{.clusters[0].name}' 2>/dev/null || echo "")
              if [ -n "$CLUSTER_NAME" ]; then
                echo "Setting cluster ${CLUSTER_NAME} --insecure-skip-tls-verify=true (dev fallback)"
                $KUBECTL config set-cluster "$CLUSTER_NAME" --insecure-skip-tls-verify=true || true
              fi

              POD_NAME="kaniko-build-${BUILD_NUMBER}"
              # create kaniko pod manifest (uses nexus-docker-secret for pushing)
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
              $KUBECTL -n ${NAMESPACE} apply -f kaniko-pod.yaml

              # Wait for pod to become Ready (up to 120s). If not, show describe & events to debug.
              echo "Waiting for pod ${POD_NAME} to become Ready..."
              if ! $KUBECTL -n ${NAMESPACE} wait --for=condition=Ready pod/${POD_NAME} --timeout=120s; then
                echo "Pod did not become Ready - showing details:"
                $KUBECTL -n ${NAMESPACE} describe pod ${POD_NAME} || true
                echo "Recent events:"
                $KUBECTL -n ${NAMESPACE} get events --sort-by='.lastTimestamp' -o wide || true
                echo "Attempting to stream logs (if any) then cleaning up..."
                $KUBECTL -n ${NAMESPACE} logs ${POD_NAME} --all-containers || true
                $KUBECTL -n ${NAMESPACE} delete pod ${POD_NAME} --ignore-not-found
                exit 1
              fi

              # copy workspace into the pod (retry if needed)
              for i in 1 2 3 4 5; do
                if $KUBECTL -n ${NAMESPACE} cp workspace.tar.gz ${POD_NAME}:/workspace/workspace.tar.gz 2>/dev/null; then
                  echo "Copied workspace.tar.gz to pod"
                  break
                else
                  echo "kubectl cp failed, retrying ($i)..."
                  sleep 2
                fi
              done

              echo "Streaming kaniko logs (build+push) — this will block until complete:"
              $KUBECTL -n ${NAMESPACE} logs -f ${POD_NAME} -c kaniko || true

              echo "Cleaning up kaniko pod..."
              $KUBECTL -n ${NAMESPACE} delete pod ${POD_NAME} --ignore-not-found || true
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
            export KUBECONFIG=$WORKSPACE/.kube/config
            KUBECTL="$WORKSPACE/.local/bin/kubectl --kubeconfig=$KUBECONFIG"

            # Update deployment image (best-effort)
            echo "Trying to update deployment image (if deployment exists)..."
            $KUBECTL -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record || true

            # If k8s manifest exists in repo, apply it; else create a minimal fallback manifest and apply.
            if [ -f k8s/frontend-deployment.yaml ]; then
              echo "Applying repo manifest k8s/frontend-deployment.yaml"
              $KUBECTL -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
            else
              echo "k8s/frontend-deployment.yaml not found — creating minimal fallback manifest and applying"
              mkdir -p k8s
              cat > k8s/frontend-deployment.yaml <<YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: ${IMAGE}:${TAG}
          ports:
            - containerPort: 80
YAML
              $KUBECTL -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
            fi
          '''
        }
      }
    }
  }

  post {
    success { echo "Deployed ${IMAGE}:${TAG}" }
    failure { echo "Pipeline failed — check console output" }
  }
}
