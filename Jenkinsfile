pipeline {
  agent {
    kubernetes {
      label "jnlp-node-${env.BUILD_NUMBER}"
      defaultContainer "node"
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent-pod
spec:
  serviceAccountName: default
  restartPolicy: Never
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
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent
    - name: jnlp
      image: jenkins/inbound-agent:latest
      args:
        - \$(JENKINS_SECRET)
        - \$(JENKINS_NAME)
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent
  volumes:
    - name: workspace-volume
      emptyDir: {}
"""
    }
  }

  environment {
    IMAGE = "nexus:5000/chandarvansh/frontend"
    TAG = "${env.BUILD_NUMBER}"
    NAMESPACE = "dev-cicd"
    WORKSPACE_DIR = "${env.WORKSPACE}"
  }

  stages {
    stage('Checkout SCM') { steps { checkout scm } }

    stage('Use Nexus npm proxy') {
      steps {
        sh "echo 'registry=http://nexus:8081/repository/npm-proxy/' > .npmrc"
        sh "cat .npmrc"
      }
    }

    stage('Install & Build') {
      steps {
        sh 'npm ci'
        sh 'npm run build -- --configuration production'
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
        sh '''
          set -e
          mkdir -p $WORKSPACE/.local/bin
          STABLE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
          curl -L -o /tmp/kubectl "https://dl.k8s.io/release/${STABLE}/bin/linux/amd64/kubectl"
          chmod +x /tmp/kubectl
          mv /tmp/kubectl $WORKSPACE/.local/bin/kubectl
          export PATH=$WORKSPACE/.local/bin:$PATH
          $WORKSPACE/.local/bin/kubectl --kubeconfig=$WORKSPACE/.kube/config version --client || true
        '''
      }
    }

    stage('Build & Package for Kaniko') {
      steps {
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

    stage('Build & Push image (Kaniko)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            set -eu
            export KUBECONFIG=$WORKSPACE/.kube/config
            KUBECTL="$WORKSPACE/.local/bin/kubectl --kubeconfig=$KUBECONFIG"

            CLUSTER_NAME=$($KUBECTL config view -o jsonpath='{.clusters[0].name}' 2>/dev/null || echo "")
            if [ -n "$CLUSTER_NAME" ]; then
              $KUBECTL config set-cluster "$CLUSTER_NAME" --insecure-skip-tls-verify=true || true
            fi

            POD_NAME="kaniko-build-${BUILD_NUMBER}"

            # Kaniko pod manifest with WAIT logic: kaniko container will wait for workspace.tar.gz before running executor
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
    command:
      - /bin/sh
      - -c
    args:
      - |
        echo "kaniko container started; waiting for /workspace/workspace.tar.gz ..."
        # wait (timeout) for the workspace tar to be copied in by Jenkins (kubectl cp)
        MAX_WAIT=120
        waited=0
        while [ ! -f /workspace/workspace.tar.gz ]; do
          if [ "$waited" -ge "$MAX_WAIT" ]; then
            echo "Timed out waiting for workspace.tar.gz after ${MAX_WAIT}s"
            exit 1
          fi
          echo "waiting for workspace.tar.gz ($waited)s..."
          sleep 1
          waited=$((waited+1))
        done
        echo "workspace.tar.gz found; extracting and running kaniko..."
        # extract tar so kaniko can see Dockerfile and context files if needed
        tar -C /workspace -xzf /workspace/workspace.tar.gz || true
        exec /kaniko/executor --context=tar://workspace/workspace.tar.gz --dockerfile=/workspace/Dockerfile --destination=IMAGE_PLACEHOLDER:TAG_PLACEHOLDER --skip-tls-verify=true
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

            echo "Waiting for kaniko pod to be scheduled..."
            $KUBECTL -n ${NAMESPACE} wait --for=condition=PodScheduled pod/${POD_NAME} --timeout=60s || true

            # Copy workspace.tar.gz into the pod (retry a few times)
            for i in 1 2 3 4 5; do
              if $KUBECTL -n ${NAMESPACE} cp workspace.tar.gz ${POD_NAME}:/workspace/workspace.tar.gz 2>/dev/null; then
                echo "Copied workspace.tar.gz to pod"
                break
              else
                echo "kubectl cp failed, retrying ($i)..."
                sleep 2
              fi
            done

            echo "Streaming kaniko logs (build+push) — blocking until complete:"
            $KUBECTL -n ${NAMESPACE} logs -f ${POD_NAME} -c kaniko || true

            echo "Cleaning up kaniko pod..."
            $KUBECTL -n ${NAMESPACE} delete pod ${POD_NAME} --ignore-not-found || true
          '''
        }
      }
    }

    stage('Deploy to cluster') {
      steps {
        sh '''
          set -eu
          export KUBECONFIG=$WORKSPACE/.kube/config
          KUBECTL="$WORKSPACE/.local/bin/kubectl --kubeconfig=$KUBECONFIG"

          $KUBECTL -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record || true

          if [ -f k8s/frontend-deployment.yaml ]; then
            $KUBECTL -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
          else
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

  post {
    success { echo "Deployed ${IMAGE}:${TAG}" }
    failure { echo "Pipeline failed — check console output" }
  }
}
