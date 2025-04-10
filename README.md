# Keyboard Deploy - Página Web Estática con CI/CD

Este proyecto consiste en una página web estática que integro con un pipeline CI/CD en Jenkins. Me propuse automatizar todo el flujo de análisis de calidad y despliegue, conectando GitHub, SonarQube y un servidor NGINX remoto. La idea es mantener el código limpio y desplegarlo en segundos después de cada push al repositorio.

---

## 🚀 ¿Qué hace este proyecto?

1. **Escucha webhooks desde GitHub**  
   Al hacer push en el repositorio, Jenkins recibe un webhook que dispara el pipeline automáticamente.

2. **Analiza el código con SonarQube**  
   El pipeline incluye un análisis estático utilizando SonarQube. Esto asegura que la calidad del código se mantenga antes de pasar a producción.

3. **Evalúa el Quality Gate**  
   Si el análisis pasa el Quality Gate configurado en SonarQube, el pipeline continúa. Si no, el despliegue se aborta automáticamente.

4. **Despliega en un servidor NGINX remoto**  
   Utilizo `sshpass` y credenciales protegidas en Jenkins para transferir los archivos al servidor y reemplazar el contenido del sitio de forma segura.

---

## 🛠️ Jenkinsfile

```groovy
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
        withSonarQubeEnv('SonarQube') {
          script {
            def scannerHome = tool 'SonarQube Scanner'
            sh "${scannerHome}/bin/sonar-scanner -X"
          }
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
            which sshpass || (sudo apt-get update && sudo apt-get install -y sshpass)

            sshpass -p "$SSH_PASSWORD" scp -o StrictHostKeyChecking=no -r * "$SSH_USER@$NGINX_IP:/tmp/web-deploy/"

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
```

---

## 📊 Configuración de SonarQube

```properties
# sonar-project.properties
sonar.projectKey=keyboarddeploy
sonar.projectName=Keyboard Deploy
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.javascript.node.maxspace=512
sonar.nodejs.executable=node
sonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/*.test.js
sonar.nodejs.executable=/usr/bin/node
```

---

## ✅ Requisitos

- Jenkins con plugins de GitHub y SonarQube
- SonarQube configurado con el proyecto `keyboarddeploy`
- Credenciales SSH configuradas en Jenkins (`nginx-deploy-creds`)
- Servidor NGINX accesible vía IP
- `sshpass` instalado o permisos para instalarlo

---

## 📁 Estructura esperada en el servidor remoto

El contenido del sitio se despliega en:

```
/var/www/html/KeyboardDeploy/
```

Si la carpeta no existe, el script la crea automáticamente.

---

## 📌 Notas adicionales

- El pipeline depende del valor de `NGINX_IP`, que puede ser inyectado por parámetros de Jenkins o definido como variable de entorno global.
- Todo el contenido de producción se borra y reemplaza con cada despliegue, así que es importante mantener respaldo si es necesario.
- Para mantener la seguridad, las credenciales se gestionan con el sistema interno de Jenkins y no se incluyen en el repositorio.

---

Este flujo me permite mantener la calidad del código y automatizar el ciclo de vida de despliegue con herramientas robustas y confiables.

---
