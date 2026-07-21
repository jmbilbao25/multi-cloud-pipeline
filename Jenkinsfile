pipeline {
  agent any

  environment {
    AZ_ACCOUNT   = 'pocstore9194'
    AZ_SHARE     = 'webcontent'
    STAGING_URL  = 'http://<YOUR-STAGING-URL>.azurecontainer.io'
    WEB_SERVER_1 = '<PRIVATE-IP-OF-WEB-SERVER-1>'
    WEB_SERVER_2 = '<PRIVATE-IP-OF-WEB-SERVER-2>'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Deploy to Staging (Azure ACI)') {
      steps {
        withCredentials([string(credentialsId: 'azure-storage-key', variable: 'AZ_KEY')]) {
          sh '''
            echo "=== Uploading files to Azure File Share (staging) ==="
            az storage file upload-batch \
              --account-name "$AZ_ACCOUNT" \
              --account-key "$AZ_KEY" \
              --destination "$AZ_SHARE" \
              --source . \
              --no-progress
          '''
        }
      }
    }

    stage('Test Staging') {
      steps {
        sh '''
          echo "=== Waiting 10 seconds for Azure to sync ==="
          sleep 10

          echo "=== Testing Staging Endpoint ==="
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$STAGING_URL")
          if [ "$HTTP_CODE" != "200" ]; then
            echo "FAIL: Staging returned HTTP $HTTP_CODE (expected 200)"
            exit 1
          fi
          echo "PASS: Staging returned HTTP 200"
        '''
      }
    }

    stage('Deploy to Production (AWS EC2s)') {
      steps {
        sshagent(credentials: ['web-server-ssh-key']) {
          sh '''
            echo "=== Deploying to AWS Web Server 1 ==="
            scp -o StrictHostKeyChecking=no *.html ubuntu@$WEB_SERVER_1:/var/www/html/
            
            echo "=== Deploying to AWS Web Server 2 ==="
            scp -o StrictHostKeyChecking=no *.html ubuntu@$WEB_SERVER_2:/var/www/html/
          '''
        }
      }
    }
  }
}
