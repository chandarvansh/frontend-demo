pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: frontend-pod
spec:
  containers:
  - name: node
    image: node:18-bullseye
    command:
      - cat
    tty: true
    resources:
      limits:
        cpu: "1"
        memory: "1Gi"
      requests:
        cpu: "200m"
        memory: "256Mi"
  - name: kubectl
    image: lachlanevenson/k8s-kubectl:v1.34.0
    command:
      - cat
    tty: true
    resources:
      limits:
        cpu: "200m"
        memory: "256Mi"
      requests:
        cpu: "50m"
        memory: "64Mi"
  # nothing special for volumes - Jenkins shares workspace by default
"""
    }
  }

  environment {
    IMAGE = "nexus:5000/chandarvansh/frontend"
    TAG = "${env.BUILD_NUMBER}"
    NAMESPACE = "dev-cicd"
  }

  stages {

    stage('Checkout') {
      steps {
        // checkout happens in the default jnlp container; using scm checkout from job
        checkout scm
      }
    }

    stage('Use Nexus npm proxy') {
      steps {
        container('node') {
          sh "echo 'registry=http://nexus:8081/repository/npm-proxy/' > .npmrc"
          sh "cat .npmrc"
        }
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
        // copy kubeconfig from Jenkins credential into workspace and make available to kubectl container
        withCredentials([file(credentialsId: 'jenkins-kubeconfig', variable: 'KCFG')]) {
          container('kubectl') {
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
    }

    stage('Build & Push image (Kaniko)') {
      steps {
        // package workspace in node container (fast), then use kubectl container to create kaniko pod and copy/tail logs
        container('node') {
          sh 'tar -czf workspace.tar.gz .'
        }

        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          container('kubectl') {
            sh '''
              KANIKO_POD=kaniko-build-$(date +%s)
cat > /tmp/kaniko-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ${KANIKO_POD}
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
EOF

              # apply kaniko pod manifest
              kubectl -n ${NAMESPACE} apply -f /tmp/kaniko-pod.yaml

              # wait for pod to appear
              for i in 1 2 3 4 5 6 7 8 9 10; do
                POD=$(kubectl -n ${NAMESPACE} get pods --no-headers -o custom-columns=":metadata.name" | grep kaniko-build || true)
                [ -n "$POD" ] && break || sleep 2
              done

              # copy workspace.tar.gz from the node container's workspace (shared) into the kaniko pod's /workspace
              kubectl -n ${NAMESPACE} cp $WORKSPACE/workspace.tar.gz ${POD}:/workspace/workspace.tar.gz || true

              # stream logs
              kubectl -n ${NAMESPACE} logs -f ${POD}

              # cleanup
              kubectl -n ${NAMESPACE} delete pod ${POD} --ignore-not-found
            '''
          }
        }
      }
    }

    stage('Deploy to Minikube') {
      steps {
        container('kubectl') {
          sh '''
            kubectl -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record || kubectl -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
          '''
        }
      }
    }

  }

  post {
    success {
      echo "Deployment successful → ${IMAGE}:${TAG}"
    }
    failure {
      echo "Pipeline failed"
    }
  }
}
