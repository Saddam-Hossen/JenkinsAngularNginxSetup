Here's your **updated NGINX setup guide** assuming **steps 1–2 (build and move) are done by Jenkins** automatically:

---

## ✅ Complete NGINX Setup Guide for Serving an Angular Project (with Jenkins handling build/deploy)

---

### ✅ 1. **Ensure Jenkins Builds Angular to Correct Folder**  

![Screenshot 2025-05-16 234108](https://github.com/user-attachments/assets/8886e4e7-4205-47d5-9411-7a7be76c09e2)


make sure your Jenkins pipeline or job:
```bash
      pipeline {
          agent any
      
          environment {
              PROD_HOST  = credentials('DO_HOST')
              PROD_USER  = credentials('DO_USER')
              DEPLOY_DIR = '/www/wwwroot/CITSNVN/jenkins/angular'
              BACKUP_DIR = '/www/wwwroot/CITSNVN/jenkins/angular_backup'
              BUILD_DIR  = 'dist/my-project'
          }
      
          stages {
              stage('Checkout') {
                  steps {
                      git branch: 'main', url: 'https://github.com/Saddam-Hossen/JenkinsAngularProject.git'
                  }
              }
      
              stage('Install Dependencies') {
                  steps {
                      bat 'npm install'
                  }
              }
      
              stage('Verify Files') {
                  steps {
                      bat 'dir src\\app\\environments'
                  }
              }
      
              stage('Build Angular Project') {
                  steps {
                      bat 'ng build --configuration=production'
                  }
              }
      
              stage('Backup Current Deployment') {
                  steps {
                      withCredentials([sshUserPrivateKey(credentialsId: 'DO_SSH_KEY', keyFileVariable: 'SSH_KEY')]) {
                          script {
                              def bashCmd = '''#!/bin/bash
                                  ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" ${PROD_USER}@${PROD_HOST} <<EOF
                                      echo "📦 Backing up current deployment..."
                                      rm -rf ${BACKUP_DIR}
                                      cp -r ${DEPLOY_DIR} ${BACKUP_DIR}
      EOF
                              '''
                              writeFile file: 'backup.sh', text: bashCmd
                              bat '"C:\\Program Files\\Git\\bin\\bash.exe" backup.sh'
                          }
                      }
                  }
              }
      
              stage('Deploy to DigitalOcean') {
                  steps {
                      withCredentials([sshUserPrivateKey(credentialsId: 'DO_SSH_KEY', keyFileVariable: 'SSH_KEY')]) {
                          script {
                              def deployCmd = """
                                  "C:\\Program Files\\Git\\bin\\bash.exe" -c \
                                  "scp -o StrictHostKeyChecking=no -i \\"$SSH_KEY\\" -r ${BUILD_DIR}/* ${PROD_USER}@${PROD_HOST}:${DEPLOY_DIR}"
                              """
                              bat deployCmd
                          }
                      }
                  }
              }
      
              stage('Reload NGINX & Restart Backend') {
                  steps {
                      withCredentials([sshUserPrivateKey(credentialsId: 'DO_SSH_KEY', keyFileVariable: 'SSH_KEY')]) {
                          script {
                              def bashCmd = '''#!/bin/bash
                                  ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" ${PROD_USER}@${PROD_HOST} <<EOF
                                      echo "🔁 Testing NGINX config..."
                                      nginx -t && systemctl reload nginx
      EOF
                              '''
                              writeFile file: 'remote.sh', text: bashCmd
                              bat '"C:\\Program Files\\Git\\bin\\bash.exe" remote.sh'
                          }
                      }
                  }
              }
          }
      
          post {
              success {
                  echo '✅ Angular build and deploy complete!'
              }
      
              failure {
                  script {
                      echo '❌ Build or deployment failed. Starting rollback...'
      
                      withCredentials([sshUserPrivateKey(credentialsId: 'DO_SSH_KEY', keyFileVariable: 'SSH_KEY')]) {
                          def rollbackCmd = '''#!/bin/bash
                              ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" ${PROD_USER}@${PROD_HOST} <<EOF
                                  echo "⏪ Rolling back to previous deployment..."
                                  rm -rf ${DEPLOY_DIR}
                                  cp -r ${BACKUP_DIR} ${DEPLOY_DIR}
                                  nginx -t && systemctl reload nginx
      EOF
                          '''
                          writeFile file: 'rollback.sh', text: rollbackCmd
                          bat '"C:\\Program Files\\Git\\bin\\bash.exe" rollback.sh'
                      }
      
                      mail to: '01957098631a@gmail.com',
                          subject: "❌ Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                          body: """\
      Hello,
      
      The Jenkins build for job *${env.JOB_NAME}* (build #${env.BUILD_NUMBER}) has **failed**.
      
      Rollback has been triggered. Please review the console output.
      
      🔗 Link: ${env.BUILD_URL}
      
      Regards,  
      Jenkins
      """
                  }
              }
          }
      }

```


---

### ✅ 2. **Create the NGINX Configuration File**

Create a config file for your site:

```bash
sudo nano /www/server/panel/vhost/nginx/sscsnvn.conf
```

Paste this:

```nginx
server {
    listen 3085;
    server_name 159.89.172.251;  # Replace with your domain if you have one

    root /www/wwwroot/CITSNVN/jenkins/angular/browser;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to Spring Boot backend on port 3086
    location /api/ {
        proxy_pass http://localhost:3086/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

### ✅ 3. **Enable the NGINX Config**

Link it to the NGINX enabled sites:

```bash
sudo ln -sf /www/server/panel/vhost/nginx/sscsnvn.conf /etc/nginx/sites-enabled/
```

---

### ✅ 4. **Test the NGINX Configuration**

Make sure the config is valid:

```bash
sudo nginx -t
```

Output should include:

```
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### ✅ 5. **Reload NGINX**

Apply changes:

```bash
sudo systemctl reload nginx
```

---

### ✅ 6. **Verify Frontend and API Are Working**

Go to:

* Angular frontend: [http://159.89.172.251:3085/](http://159.89.172.251:3085/)
* Spring Boot API: [http://159.89.172.251:3085/api/student](http://159.89.172.251:3085/api/student)

---

### ✅ 7. **Firewall Check (if needed)**

Ensure port `3085` is allowed:

```bash
sudo ufw allow 3085
```

---

### 🔍 Troubleshooting

| Issue                      | Solution                                                              |
| -------------------------- | --------------------------------------------------------------------- |
| `Website not found`        | Check Jenkins deployed build to `browser/`, check `index.html` exists |
| API not working            | Ensure Spring Boot is running on `localhost:3086`                     |
| NGINX errors               | Run `sudo nginx -t` and check `/var/log/nginx/error.log`              |
| Blank page / Routing issue | Confirm `try_files $uri $uri/ /index.html;` is in the config          |

---

Let me know if you want to **auto-reload NGINX after Jenkins deploy** or hook in backend restarts too!
