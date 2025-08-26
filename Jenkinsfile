pipeline {
  agent { label 'azure-vm' }           // VM Azure registrata come nodo Jenkins
  options { timestamps() }

  parameters {
    choice(name: 'TRIGGER_TYPE', choices: ['push', 'single'], description: 'push=deploy completo; single=solo un servizio')
    string(name: 'SERVICE_NAME', defaultValue: 'assignment', description: 'Usato se TRIGGER_TYPE=single')
    string(name: 'ROLLOUT_TIMEOUT', defaultValue: '180s',    description: 'Timeout per kubectl rollout status')
  }

  environment {
    MK8S            = '/snap/bin/microk8s'
    K8S_DIR         = 'k8s'
    DOCKERHUB_CREDS = 'dockerhub-creds'
    REGISTRY        = 'ale175'
    NS_LIST         = 'assignment review user-manager submission report'
  }

  stages {
    stage('Init') {
      steps {
        script {
          env.MODE = params.TRIGGER_TYPE
          env.SVC  = params.SERVICE_NAME
          env.TAG  = params.IMAGE_TAG
          echo "MODE=${env.MODE} | SERVICE=${env.SVC} | TAG=${env.TAG}"
        }
      }
    }

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Prerequisiti VM') {
      steps {
        sh """
          set -e
          which docker
          ${MK8S} status --wait-ready
          ${MK8S} kubectl version --client=true
        """
      }
    }

    stage('Docker login (opzionale)') {
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', DOCKERHUB_CREDS) {
            echo 'Login al registry effettuato'
          }
        }
      }
    }

    stage('Apply manifest') {
      steps {
        sh """
          set -e
          if [ "${MODE}" = "push" ]; then
            ${MK8S} kubectl apply -f "${K8S_DIR}/namespaces.yaml"
            ${MK8S} kubectl apply -R -f "${K8S_DIR}/databases"
            ${MK8S} kubectl apply -R -f "${K8S_DIR}/services"
            ${MK8S} kubectl apply -f "${K8S_DIR}/config.yaml"
            ${MK8S} kubectl apply -f "${K8S_DIR}/secrets.yaml"
            ${MK8S} kubectl apply -f "${K8S_DIR}/ingress.yaml"
          else
            ${MK8S} kubectl apply -R -f "${K8S_DIR}/services/${SVC}"
          fi
        """
      }
    }

    stage('Attendi rollout (usa la strategia dei manifest)') {
      steps {
        script {
          if (env.MODE == 'push') {
            sh '''
              set -e
              for ns in ${NS_LIST}; do
                deps=$(${MK8S} kubectl -n "$ns" get deploy -o name || true)
                [ -z "$deps" ] && { echo "Nessun deployment in $ns"; continue; }
                echo "$deps" | while read -r d; do
                  echo "Rollout $d in $ns"
                  ${MK8S} kubectl -n "$ns" rollout status "$d" --timeout=${ROLLOUT_TIMEOUT}
                done
              done
            '''
          } else {
            sh '''
              set -e
              ns="${SVC}"
              d="deploy/${SVC}"
              ${MK8S} kubectl -n "$ns" rollout status "$d" --timeout=${ROLLOUT_TIMEOUT}
            '''
          }
        }
      }
    }
  }

  post {
    success { echo "Deploy OK — MODE=${env.MODE}, SERVICE=${env.SVC}, TAG=${env.TAG}" }
    failure { echo "Deploy fallito — controlla i log" }
  }
}
