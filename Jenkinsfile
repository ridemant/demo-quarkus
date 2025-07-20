pipeline {
  agent any
  options {
    cleanWs()
  }

  environment {
    SSH_KEY_ID   = 'vps-ssh'
    VPS_TARGET   = 'root@167.86.115.24'
    IMAGE_NAME   = 'quarkus-app-dev:latest'
    CONTAINER    = 'quarkus-dev'
    REMOTE_PORT  = '9081'
  }

  tools {
    jdk 'jdk17'
  }

  stages {
    stage('Clonar') {
      steps {
        git 'https://github.com/ridemant/demo-quarkus.git'
      }
    }

    stage('Compilar') {
      steps {
        sh './mvnw clean package -DskipTests'
      }
    }

    stage('Build Imagen') {
      steps {
        sh "docker build -t ${IMAGE_NAME} ."
      }
    }

    stage('Enviar al VPS') {
      steps {
        sshagent([SSH_KEY_ID]) {
          sh """
            docker save ${IMAGE_NAME} | bzip2 | ssh -o StrictHostKeyChecking=no ${VPS_TARGET} 'bunzip2 | podman load'
          """
        }
      }
    }

    stage('Desplegar en Podman') {
      steps {
        sshagent([SSH_KEY_ID]) {
          sh """
            ssh -o StrictHostKeyChecking=no ${VPS_TARGET} '
              podman stop ${CONTAINER} || true
              podman rm ${CONTAINER} || true
              podman run -d --name ${CONTAINER} -p ${REMOTE_PORT}:8080 ${IMAGE_NAME}
            '
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Despliegue listo en: http://${VPS_TARGET.split('@')[1]}:${REMOTE_PORT}"
    }
  }
}
