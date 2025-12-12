pipeline {
  agent any
  environment {
    IMAGE = "nexus:5000/chandarvansh/frontend"
    TAG = "${env.BUILD_NUMBER}"
    NAMESPACE = "dev-cicd"
  }
  stages {
    stage('Checkout') {
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
        sh 'npm ci'
        sh 'npm run build -- --configuration production'
      }
    }
    stage('Prepare kubeconfig') {
      steps {
        withCredentials([file(credentialsId: 'jenkins-kubeconfig', variable: 'KCFG')]) {
          sh '''
            mkdir -p $HOME/.kube
            cp "$KCFG" $HOME/.kube/config
            chmod 600 $HOME/.kube/config
            kubectl config current-context || true
          '''
        }
      }
    }
    stage('Build & Push image (Kaniko)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            # package workspace for Kaniko
            tar -czf workspace.tar.gz .

            # pod manifest for kaniko that uses nexus-docker-secret for docker auth
            cat > kaniko-pod.yaml <<YAML
apiVersion: v1
kind: Pod
metadata:
  name: kaniko-build-\$(date +%s)
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

            # create pod, copy context, run and stream logs
            kubectl -n ${NAMESPACE} apply -f kaniko-pod.yaml
            sleep 2
            PODNAME=$(kubectl -n ${NAMESPACE} get pods --no-headers -o custom-columns=":metadata.name" | grep kaniko-build || true)
            # fallback wait
            for i in 1 2 3 4 5; do
              PODNAME=$(kubectl -n ${NAMESPACE} get pods --no-headers -o custom-columns=":metadata.name" | grep kaniko-build || true)
              [ -n "$PODNAME" ] && break || sleep 2
            done
            kubectl -n ${NAMESPACE} cp workspace.tar.gz ${PODNAME}:/workspace/workspace.tar.gz || true
            kubectl -n ${NAMESPACE} logs -f ${PODNAME}
            kubectl -n ${NAMESPACE} delete pod/${PODNAME} || true
          '''
        }
      }
    }
    stage('Deploy to Minikube') {
      steps {
        sh '''
          kubectl -n ${NAMESPACE} set image deployment/frontend-deployment frontend=${IMAGE}:${TAG} --record || kubectl -n ${NAMESPACE} apply -f k8s/frontend-deployment.yaml
        '''
      }
    }
  }
  post {
    success { echo "Deployed ${IMAGE}:${TAG}" }
    failure { echo "Pipeline failed" }
  }
}
