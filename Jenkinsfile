pipeline {
  agent { label 'ansible-agent' }  // ensure this agent has ansible & curl installed
  parameters {
    string(name: 'ARTIFACT_VERSION', defaultValue: '1.0.0', description: 'Artifact version to build/deploy')
    choice(name: 'DEPLOY_ENV', choices: ['staging','production'], description: 'Target environment for deploy')
  }
  environment {
    INVENTORY = "hosts.ini"
    ANSIBLE_PLAYBOOK = "deploy-app.yml"
    SSH_KEY_CREDENTIALS = 'jenkins-ssh-key-id'       // SSH private key credential (privateKey)
    MYSQL_ROOT = credentials('mysql-root-creds-id')  // username/password or secret text
    // Nexus credentials: must be configured in Jenkins as "Username with password" type
    NEXUS_CRED = credentials('nexus-user-id')        // provides NEXUS_CRED_USR and NEXUS_CRED_PSW
    // Nexus base URL (no trailing slash), and repository path where uploads should go:
    NEXUS_URL = "http://3.125.120.16:8081/nexus/content/"
    NEXUS_REPO_PATH = "repository/releases/php-crud-app"   // e.g. repository/releases/<repoName>
    //AWS_CREDENTIALS = credentials('aws-creds-id')    // if using S3 (optional)
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Test') {
      steps {
        // For a simple PHP app, we just archive files. If composer/phpunit needed, run here.
        sh 'php -v || true'
        sh 'echo "Running simple lint"'
        sh 'php -l roles/deploy_app/files/index.php || true'
      }
    }

    stage('Archive Artifact') {
      steps {
        script {
          // Create a zip artifact
          sh "mkdir -p artifacts && zip -r artifacts/phpapp-${params.ARTIFACT_VERSION}.zip roles/deploy_app/files/*"

          // Upload to S3 (optional) - retained for reference
          //withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds-id']]) {
            // Uncomment if you want to upload to S3 as well
            // sh "aws s3 cp artifacts/phpapp-${params.ARTIFACT_VERSION}.zip s3://my-artifact-bucket/phpapp-${params.ARTIFACT_VERSION}.zip"


          // Upload to Nexus (using username/password credentials stored in Jenkins)
          // Jenkins credentials('nexus-user-id') used in environment block exposes:
          // NEXUS_CRED_USR and NEXUS_CRED_PSW
          def artifactFile = "artifacts/phpapp-${params.ARTIFACT_VERSION}.zip"
          def nexusUploadUrl = "${NEXUS_URL}/${NEXUS_REPO_PATH}/${params.ARTIFACT_VERSION}/phpapp-${params.ARTIFACT_VERSION}.zip"

          echo "Uploading ${artifactFile} to Nexus at ${nexusUploadUrl}"

          // Try upload with curl and simple retry logic
          sh """
            set -euo pipefail
            RETRIES=3
            SLEEP=3
            for i in \$(seq 1 \$RETRIES); do
              echo \"Attempt \$i: uploading to Nexus...\"
              HTTP_CODE=\$(curl -sS -u \"${NEXUS_CRED_USR}:${NEXUS_CRED_PSW}\" -w '%{http_code}' --upload-file ${artifactFile} "${nexusUploadUrl}" -o /dev/null || echo "000")
              echo "HTTP_CODE=\$HTTP_CODE"
              if [ "\$HTTP_CODE" = "201" ] || [ "\$HTTP_CODE" = "200" ]; then
                echo "Upload successful (HTTP \$HTTP_CODE)"
                break
              else
                echo "Upload attempt \$i failed with HTTP \$HTTP_CODE"
                if [ \$i -lt \$RETRIES ]; then
                  echo "Retrying in \$SLEEP seconds..."
                  sleep \$SLEEP
                else
                  echo "All upload attempts failed."
                  exit 1
                fi
              fi
            done
          """
        }
      }
    }

    // --- NEW STAGE: Download artifact from Nexus (added here) ---
    stage('Download Artifact from Nexus') {
      steps {
        script {
          def downloadUrl = "${NEXUS_URL}/${NEXUS_REPO_PATH}/${params.ARTIFACT_VERSION}/phpapp-${params.ARTIFACT_VERSION}.zip"
          def outFile = "artifacts/phpapp-${params.ARTIFACT_VERSION}.zip"
          echo "Downloading artifact from Nexus: ${downloadUrl} -> ${outFile}"

          // Use username/password credential to download
          withCredentials([usernamePassword(credentialsId: 'nexus-user-id', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PSW')]) {
            sh """
              set -euo pipefail
              mkdir -p artifacts
              RETRIES=3
              SLEEP=3
              for i in \$(seq 1 \$RETRIES); do
                echo "Attempt \$i: downloading from Nexus..."
                HTTP_CODE=\$(curl -sS -u "${NEXUS_USER}:${NEXUS_PSW}" -w '%{http_code}' -o ${outFile} "${downloadUrl}" || echo "000")
                echo "HTTP_CODE=\$HTTP_CODE"
                if [ "\$HTTP_CODE" = "200" ]; then
                  echo "Download successful (HTTP \$HTTP_CODE)"
                  break
                else
                  echo "Download attempt \$i failed with HTTP \$HTTP_CODE"
                  if [ \$i -lt \$RETRIES ]; then
                    echo "Retrying in \$SLEEP seconds..."
                    sleep \$SLEEP
                  else
                    echo "All download attempts failed."
                    exit 1
                  fi
                fi
              done
            """
          }
        }
      }
    }

    stage('Deploy to Staging') {
      when {
        expression { params.DEPLOY_ENV == 'staging' || params.DEPLOY_ENV == 'production' }
      }
      steps {
        // Run Ansible to deploy to staging (if DEPLOY_ENV==staging) or production (for manual promotion)
        withCredentials([sshUserPrivateKey(credentialsId: env.SSH_KEY_CREDENTIALS, keyFileVariable: 'SSH_KEY')]) {
          sh '''
            export ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i ${INVENTORY} ${ANSIBLE_PLAYBOOK} -e target_group=${DEPLOY_ENV} \
               -e mysql_root_password=${MYSQL_ROOT_PW} \
               -e mysql_app_password='app_password_here' \
               -e artifact_path=${WORKSPACE}/artifacts/phpapp-${ARTIFACT_VERSION}.zip
          '''
        }
      }
    }

    stage('Manual Approval to Production') {
      when {
        expression { params.DEPLOY_ENV == 'staging' }
      }
      steps {
        input message: 'Promote to production?', ok: 'Promote'
      }
    }

    stage('Promote to Production') {
      when {
        expression { params.DEPLOY_ENV == 'staging' }
      }
      steps {
        // Now deploy to production after manual approval
        withCredentials([sshUserPrivateKey(credentialsId: env.SSH_KEY_CREDENTIALS, keyFileVariable: 'SSH_KEY')]) {
          sh '''
            export ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i ${INVENTORY} ${ANSIBLE_PLAYBOOK} -e target_group=production \
               -e mysql_root_password=${MYSQL_ROOT_PW} \
               -e mysql_app_password='app_password_here' \
               -e artifact_path=${WORKSPACE}/artifacts/phpapp-${ARTIFACT_VERSION}.zip
          '''
        }
      }
    }
  } // stages

  //post {
    //always {
      //archiveArtifacts artifacts: 'artifacts/*.zip', fingerprint: true
   // }
 // }
}
