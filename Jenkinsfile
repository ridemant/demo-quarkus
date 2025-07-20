pipeline {
  agent any

  environment {
    SSH_KEY_ID   = 'vps-ssh'
    VPS_TARGET   = 'root@167.86.115.24'
    IMAGE_NAME   = 'quarkus-app-dev:latest'
    CONTAINER    = 'quarkus-dev'
    REMOTE_PORT  = '9092'
  }

  tools {
    jdk 'jdk17'
  }

  stages {
    stage('Compilar') {
      steps {
        sh '''
          chmod +x mvnw
          ./mvnw clean package -DskipTests "-Dquarkus.package.type=uber-jar"
        '''
      }
    }

    stage('Enviar al VPS y construir imagen') {
      steps {
        sshagent([SSH_KEY_ID]) {
          sh '''
            JAR_PATH=$(ls target/*-runner.jar)
            JAR_NAME=$(basename $JAR_PATH)

            echo "📤 Subiendo $JAR_NAME al VPS..."
            ssh -o StrictHostKeyChecking=no $VPS_TARGET 'rm -rf /tmp/quarkus-build && mkdir -p /tmp/quarkus-build'
            scp $JAR_PATH $VPS_TARGET:/tmp/quarkus-build/$JAR_NAME

            echo "🔧 Creando Dockerfile y construyendo imagen..."
            ssh $VPS_TARGET "
              cd /tmp/quarkus-build &&
              echo 'FROM eclipse-temurin:17' > Dockerfile &&
              echo 'COPY $JAR_NAME app.jar' >> Dockerfile &&
              echo 'ENTRYPOINT [\\"java\\", \\"-Dquarkus.http.port=8081\\", \\"-jar\\", \\"app.jar\\"]' >> Dockerfile &&
              podman build -t $IMAGE_NAME .
            "
          '''
        }
      }
    }

    stage('Desplegar en Podman') {
      steps {
        sshagent([SSH_KEY_ID]) {
          sh '''
            echo "🚀 Desplegando en Podman..."
            ssh $VPS_TARGET "
              podman stop $CONTAINER || true &&
              podman rm $CONTAINER || true &&
              podman run -d --name $CONTAINER --network=host $IMAGE_NAME
            "
          '''
        }
      }
    }
  }

  post {
    always {
      echo "🧹 Limpiando artefactos locales..."
      sh '''
        rm -rf target
        rm -rf *.tar *.gz *.zip || true
      '''
    }
  }
}
