pipeline {
  agent any
  environment {
        NGINX_IP = "${env.NGINX_IP}"
  }
  stages {

    stage('Hola GitHub') {
      steps {
        echo "✅ Webhook recibido desde GitHub 🎉"
      }
    }
    
    stage('SonarQube analysis') {
      steps {
        script {
          scannerHome = tool 'SonarQube Scanner'
        }
        withSonarQubeEnv('SonarQube') {
          sh "${scannerHome}/bin/sonar-scanner"
        }
      }
    }
   
    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
    stage('Deploy') {
        
        steps {
            withCredentials([usernamePassword(
                credentialsId: 'nginx-deploy-creds',
                usernameVariable: 'SSH_USER',
                passwordVariable: 'SSH_PASSWORD'
            )]) {
                sh '''
                    # Instalar sshpass si no está presente (requiere sudo)
                    which sshpass || (sudo apt-get update && sudo apt-get install -y sshpass)
                    
                    # Copiar archivos al servidor remoto
                    sshpass -p "$SSH_PASSWORD" scp -o StrictHostKeyChecking=no -r * "$SSH_USER@$NGINX_IP:/tmp/web-deploy/"
                    
                    # Ejecutar comandos remotos
                    sshpass -p "$SSH_PASSWORD" ssh -tt "$SSH_USER@$NGINX_IP" '
                        sudo rm -rf /var/www/html/KeyboardDeploy/*
                        sudo cp -r /tmp/web-deploy/* /var/www/html/KeyboardDeploy/
                    '
                '''
  
            }
        }
      }
  }
}
