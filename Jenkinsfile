pipeline {

  agent any

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

    stage('Compilar') {
      steps {
        sh '''
          chmod +x mvnw
          ./mvnw clean package -DskipTests
        '''
      }
    }

    stage('Enviar al VPS y construir imagen') {
      steps {
        sshagent([SSH_KEY_ID]) {
          sh """
            echo "📤 Subiendo proyecto al VPS..."
            ssh -o StrictHostKeyChecking=no ${VPS_TARGET} 'rm -rf /tmp/quarkus-build && mkdir -p /tmp/quarkus-build'
            scp -r Jenkinsfile README.md mvnw mvnw.cmd pom.xml src target ${VPS_TARGET}:/tmp/quarkus-build

            ssh ${VPS_TARGET} '
              cd /tmp/quarkus-build &&
              echo "🔧 Creando Dockerfile temporal..." &&
              JAR_NAME=\$(basename \$(ls target/*.jar | head -n1)) &&
              echo "FROM docker.io/eclipse-temurin:17" > Dockerfile &&
              echo "COPY target/\$JAR_NAME app.jar" >> Dockerfile &&
              echo "ENTRYPOINT [\\"java\\", \\"-jar\\", \\"app.jar\\"]" >> Dockerfile &&
              podman build -t ${IMAGE_NAME} .
            '
          """
        }
      }
    }

    stage('Desplegar en Podman') {
      steps {
        sshagent([SSH_KEY_ID]) {
          sh """
            echo "🚀 Desplegando en Podman..."
            ssh ${VPS_TARGET} '
              podman stop ${CONTAINER} || true &&
              podman rm ${CONTAINER} || true &&
              podman run -d --name ${CONTAINER} -p ${REMOTE_PORT}:8080 ${IMAGE_NAME}
            '
          """
        }
      }
    }
  }

  post {
    always {
      echo "🧹 Limpiando workspace final..."
      sh '''
        rm -rf target
        rm -rf *.tar *.gz *.zip || true
      '''
    }
    success {
      echo "✅ Despliegue listo en: http://${VPS_TARGET.split('@')[1]}:${REMOTE_PORT}"
    }
  }
}
