pipeline {
  agent any

  environment {
    AZ_ACCOUNT   = 'pocstore9194'
    AZ_SHARE     = 'webcontent'
    STAGING_URL  = 'http://poc-nginx-9194.eastus.azurecontainer.io'
    WEB_SERVER_1 = '10.0.3.51'
    WEB_SERVER_2 = '10.0.4.175'
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
            scp -o StrictHostKeyChecking=no *.html ubuntu@$WEB_SERVER_1:/home/ubuntu/
            ssh -o StrictHostKeyChecking=no ubuntu@$WEB_SERVER_1 'sudo cp /home/ubuntu/*.html /var/www/html/'
            
            echo "=== Deploying to AWS Web Server 2 ==="
            scp -o StrictHostKeyChecking=no *.html ubuntu@$WEB_SERVER_2:/home/ubuntu/
            ssh -o StrictHostKeyChecking=no ubuntu@$WEB_SERVER_2 'sudo cp /home/ubuntu/*.html /var/www/html/'
          '''
        }
      }
    }
