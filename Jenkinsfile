// pipeline {
//   agent { label 'ansible-agent' }  // ensure this agent has ansible & curl installed
//   parameters {
//     string(name: 'ARTIFACT_VERSION', defaultValue: '1.0.0', description: 'Artifact version to deploy')
//     choice(name: 'DEPLOY_ENV', choices: ['staging','production'], description: 'Target environment for deploy')
//   }
//   environment {
//     INVENTORY = "hosts.ini"
//     ANSIBLE_PLAYBOOK = "deploy-app.yml"
//     SSH_KEY_CREDENTIALS = 'jenkins-ssh-key-id'       // SSH private key credential (privateKey)
//     MYSQL_ROOT = credentials('mysql-root-creds-id')  // username/password or secret text
//     // Nexus credentials: must be configured in Jenkins as "Username with password" type
//     NEXUS_CRED = credentials('nexus-user-id')        // provides NEXUS_CRED_USR and NEXUS_CRED_PSW
//     // Nexus base URL (no trailing slash), and repository path where uploads should go:
//     NEXUS_URL = "http://3.123.189.95:8081/nexus/content"
//     NEXUS_REPO_PATH = "repositories/releases/php-crud-app"   // e.g. repository/releases/<repoName>
//     //AWS_CREDENTIALS = credentials('aws-creds-id')    // if using S3 (optional)
//   }

//   stages {
//     stage('Checkout') {
//       steps {
//         checkout scm
//       }
//     }

//     stage('Build & Test') {
//       steps {
//         // For a simple PHP app, we just archive files. If composer/phpunit needed, run here.
//         sh 'php -v || true'
//         sh 'echo "Running simple lint"'
//         sh 'php -l deploy_app/files/index.php || true'
//       }
//     }

//     stage('Archive Artifact') {
//       steps {
//         script {
//           // Create a zip artifact
//           sh "mkdir -p artifacts && zip -r artifacts/phpapp-${params.ARTIFACT_VERSION}.zip deploy_app/files/*"
          
//           //sh "mkdir -p artifacts && cd deploy_app/files && zip -r artifacts/phpapp-${params.ARTIFACT_VERSION}.zip "
//         }
//       }
//     }


//     stage('Upload to Nexus') {
//       steps {
//         script {
//           // compute non-secret values in Groovy and put them into env for the shell
//           env.ARTIFACT_VERSION = params.ARTIFACT_VERSION ?: '1.0.0'
//           env.ARTIFACT_FILE = "artifacts/phpapp-${env.ARTIFACT_VERSION}.zip"
//           env.NEXUS_UPLOAD_URL = "${NEXUS_URL}/${NEXUS_REPO_PATH}/${env.ARTIFACT_VERSION}/phpapp-${env.ARTIFACT_VERSION}.zip"

//           echo "Uploading ${env.ARTIFACT_FILE} to Nexus at ${env.NEXUS_UPLOAD_URL}"

//           // Use withCredentials to inject credentials into shell env as NEXUS_USER / NEXUS_PSW
//           withCredentials([usernamePassword(credentialsId: 'nexus-user-id',
//                                        usernameVariable: 'NEXUS_USER',
//                                        passwordVariable: 'NEXUS_PSW')]) {
//           // Use a single quoted heredoc so Groovy does not interpolate secrets.
//             sh '''#!/usr/bin/env bash
//               set -euo pipefail

//               # Local copies (defensive)
//               ARTIFACT_FILE="${ARTIFACT_FILE:-artifacts/phpapp-1.0.0.zip}"
//               NEXUS_UPLOAD_URL="${NEXUS_UPLOAD_URL}"
//               RETRIES=3
//               SLEEP=3

//               echo "ARTIFACT_FILE=${ARTIFACT_FILE}"
//               echo "NEXUS_UPLOAD_URL=${NEXUS_UPLOAD_URL}"
//               echo "Checking artifact exists..."
//               if [[ ! -f "${ARTIFACT_FILE}" ]]; then
//                 echo "ERROR: artifact not found: ${ARTIFACT_FILE}"
//                 ls -la "$(dirname "${ARTIFACT_FILE}")" || true
//                 exit 2
//               fi

//               for i in $(seq 1 ${RETRIES}); do
//                 echo "Attempt ${i}: uploading to Nexus..."
//                 # upload and capture http code (curl writes body to /dev/null)
//                 HTTP_CODE=$(curl -sS -u "${NEXUS_USER}:${NEXUS_PSW}" -w '%{http_code}' --upload-file "${ARTIFACT_FILE}" "${NEXUS_UPLOAD_URL}" -o /dev/null || echo "000")
//                 echo "HTTP_CODE=${HTTP_CODE}"
//                 if [[ "${HTTP_CODE}" == "200" || "${HTTP_CODE}" == "201" || "${HTTP_CODE}" == "204" ]]; then
//                   echo "Upload successful (HTTP ${HTTP_CODE})"
//                   exit 0
//                 else
//                   echo "Upload attempt ${i} failed with HTTP ${HTTP_CODE}"
//                   if [[ ${i} -lt ${RETRIES} ]]; then
//                     echo "Retrying in ${SLEEP} seconds..."
//                     sleep ${SLEEP}
//                   else
//                     echo "All upload attempts failed."
//                     exit 1
//                   fi
//                 fi
//               done
//             '''
//           } // end withCredentials
//         } // end script
//       } // end steps
//     } // end stage



//     // --- Download artifact from Nexus ---
//     stage('Download Artifact from Nexus') {
//       steps {
//         script {
//           // Ensure ARTIFACT_VERSION is available (fallback default)
//           env.ARTIFACT_VERSION = params.ARTIFACT_VERSION ?: '1.0.0'

//           // Compute values in Groovy and export to environment for shell use
//           env.DOWNLOAD_URL = "${NEXUS_URL}/${NEXUS_REPO_PATH}/${env.ARTIFACT_VERSION}/phpapp-${env.ARTIFACT_VERSION}.zip"
//           env.OUT_FILE = "artifacts/phpapp-${env.ARTIFACT_VERSION}.zip"

//           echo "Downloading artifact from Nexus: ${env.DOWNLOAD_URL} -> ${env.OUT_FILE}"

//           withCredentials([usernamePassword(credentialsId: 'nexus-user-id',
//                                        usernameVariable: 'NEXUS_USER',
//                                        passwordVariable: 'NEXUS_PSW')]) {
//             sh '''#!/usr/bin/env bash
//               set -euo pipefail

//               # Local copies (defensive)
//               DOWNLOAD_URL="${DOWNLOAD_URL}"
//               OUT_FILE="${OUT_FILE:-artifacts/phpapp-1.0.0.zip}"
//               RETRIES=3
//               SLEEP=3

//               echo "DOWNLOAD_URL=${DOWNLOAD_URL}"
//               echo "OUT_FILE=${OUT_FILE}"

//               mkdir -p "$(dirname "${OUT_FILE}")"

//               for i in $(seq 1 ${RETRIES}); do
//                 echo "Attempt ${i}: downloading from Nexus..."
//                 # -sS for silent but show errors; -w to get HTTP code; -f isn't used so we capture HTTP code
//                 HTTP_CODE=$(curl -sS -u "${NEXUS_USER}:${NEXUS_PSW}" -w '%{http_code}' -o "${OUT_FILE}" "${DOWNLOAD_URL}" || echo "000")
//                 echo "HTTP_CODE=${HTTP_CODE}"
//                 if [[ "${HTTP_CODE}" == "200" || "${HTTP_CODE}" == "201" || "${HTTP_CODE}" == "204" ]]; then
//                   echo "Download successful (HTTP ${HTTP_CODE})"
//                   break
//                 else
//                   echo "Download attempt ${i} failed with HTTP ${HTTP_CODE}"
//                   if [[ ${i} -lt ${RETRIES} ]]; then
//                     echo "Retrying in ${SLEEP} seconds..."
//                     sleep ${SLEEP}
//                   else
//                     echo "All download attempts failed."
//                     # show what exists for debugging
//                     ls -la "$(dirname "${OUT_FILE}")" || true
//                     exit 1
//                   fi
//                 fi
//               done
//             '''
//           } // withCredentials
//         } // script
//       } // steps
//     } // stage




//     stage('Deploy to Staging') {
//       when {
//         expression { params.DEPLOY_ENV == 'staging' || params.DEPLOY_ENV == 'production' }
//       }
//       steps {
//         // Run Ansible to deploy to staging (if DEPLOY_ENV==staging) or production (for manual promotion)
//         withCredentials([sshUserPrivateKey(credentialsId: env.SSH_KEY_CREDENTIALS, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
//           sh '''
//             export ANSIBLE_HOST_KEY_CHECKING=False
//             ansible-playbook -i ${INVENTORY} ${ANSIBLE_PLAYBOOK} -e target_group=staging \
//                -e mysql_root_password=${MYSQL_ROOT_PW} \
//                --private-key "$SSH_KEY" -u "$SSH_USER" \
//                -e mysql_app_password='app_password_here' \
//                -e artifact_path=${WORKSPACE}/artifacts/phpapp-${ARTIFACT_VERSION}.zip \
//                -e artifact_version=${ARTIFACT_VERSION}

//           '''
//         }
//       }
//     }

//     stage('Manual Approval to Production') {
//       when {
//         expression { params.DEPLOY_ENV == 'staging' }
//       }
//       steps {
//         input message: 'Promote to production?', ok: 'Promote'
//       }
//     }

//     stage('Promote to Production') {
//       when {
//         expression { params.DEPLOY_ENV == 'staging' }
//       }
//       steps {
//         // Now deploy to production after manual approval
//         withCredentials([sshUserPrivateKey(credentialsId: env.SSH_KEY_CREDENTIALS, keyFileVariable: 'SSH_KEY')]) {
//           sh '''
//             export ANSIBLE_HOST_KEY_CHECKING=False
//             ansible-playbook -i ${INVENTORY} ${ANSIBLE_PLAYBOOK} -e target_group=production \
//                -e mysql_root_password=${MYSQL_ROOT_PW} \
//                -e mysql_app_password='app_password_here' \
//                -e artifact_path=${WORKSPACE}/artifacts/phpapp-${ARTIFACT_VERSION}.zip \
//                -e artifact_version=${ARTIFACT_VERSION}
//           '''
//         }
//       }
//     }
//   } // stages


//   post {
//     success { echo "Pipeline completed" }
//     failure { echo "Pipeline failed" }
//   }
//   //post {
//     //always {
//       //archiveArtifacts artifacts: 'artifacts/*.zip', fingerprint: true
//    // }
//  // }
// }











pipeline {
  agent { label 'ansible-agent' }

  environment {
      APP_NAME = "php-crud-app"
      VERSION = "${env.BUILD_NUMBER}"
      INVENTORY = "hosts.ini"
      ANSIBLE_PLAYBOOK = "deploy-app.yml"
      SSH_KEY_CREDENTIALS = 'jenkins-ssh-key-id'       // SSH private key credential (privateKey)
      MYSQL_ROOT = credentials('mysql-root-creds-id')  // username/password or secret text
      // Nexus credentials: must be configured in Jenkins as "Username with password" type
      NEXUS_CRED = credentials('nexus-user-id')        // provides NEXUS_CRED_USR and NEXUS_CRED_PSW
      // Nexus base URL (no trailing slash), and repository path where uploads should go:
      NEXUS_URL = "http://3.123.189.95:8081/nexus/content"
      NEXUS_REPO = "repositories/releases/php-crud-app"   // e.g. repository/releases/<repoName>
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }


    stage('Compute Version') {
      steps {
        script {
          // Use BUILD_NUMBER as version
          env.VERSION = "${env.BUILD_NUMBER}"
          echo "Using build-version: ${env.VERSION}"
          env.ARTIFACT_NAME = "${env.APP_NAME}-${env.VERSION}.zip"
          env.ARTIFACT_PATH = "artifacts/${env.ARTIFACT_NAME}"
          env.NEXUS_UPLOAD_URL = "${NEXUS_URL}/${NEXUS_REPO}/${env.VERSION}/${env.ARTIFACT_NAME}"
          env.NEXUS_DOWNLOAD_URL = "${NEXUS_UPLOAD_URL}"
        }
      }
    }


    stage('Build') {
      steps {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            mkdir -p artifacts
            # create flat zip of deploy_app/files (so index.php is at root of zip)
            pushd deploy_app/files
            zip -r "${WORKSPACE}/${ARTIFACT_PATH}" .
            popd
            ls -lh artifacts || true
            '''
      }
    }

    stage('Test') {
      steps {
          sh 'php -l deploy_app/files/index.php'
      }
    }


    stage('Upload to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-user-id', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PSW')]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            if [[ ! -f "${WORKSPACE}/${ARTIFACT_PATH}" ]]; then
              echo "Artifact missing: ${WORKSPACE}/${ARTIFACT_PATH}"
              exit 2
            fi
            echo "Uploading ${ARTIFACT_PATH} -> ${NEXUS_UPLOAD_URL}"
            curl --fail -S -u "${NEXUS_USER}:${NEXUS_PSW}" -T "${WORKSPACE}/${ARTIFACT_PATH}" "${NEXUS_UPLOAD_URL}"
            echo "Artifact Uploaded To Nexus."
            '''
        }
      }
    }



    stage('Assign Version to Deploy') { 
      steps {
        script {
          // Prompt user for the version to promote (manual input)
          def userInput = input(
            message: 'Enter the artifact version to deploy',
            parameters: [string(name: 'VERSION_NUMBER', defaultValue: "${env.VERSION ?: ''}")],
            ok: 'Deploy'
          )

          def v = userInput instanceof Map ? userInput.get('VERSION_NUMBER') : userInput
          if (v == null || v.toString().trim() == '') {
            error "VERSION_NUMBER REQUIRED"
          }

          env.VERSION_NUMBER = v.toString().trim()
          //env.VERSION_NUMBER = userInput['VERSION_NUMBER'].trim()
          echo "Deploying artifact version: ${env.VERSION_NUMBER}"

          // Optional sanity: verify artifact exists in Nexus using nexus creds
          withCredentials([usernamePassword(credentialsId: 'nexus-user-id', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PSW')]) {
            sh '''#!/usr/bin/env bash
              set -euo pipefail
              ART="$VERSION_NUMBER"
              ART_NAME="php-app-crud-$VERSION_NUMBER.zip"
              ART_URL="${NEXUS_URL}/${NEXUS_REPO}/$VERSION_NUMBER/"
              echo "Checking Nexus artifact: ${ART_URL}"
              HTTP=$(curl -s -o /dev/null -w '%{http_code}' -u "${NEXUS_USER}:${NEXUS_PSW}" "${ART_URL}" || echo "000")
              if [[ "$HTTP" != "200" && "$HTTP" != "201" ]]; then
                echo "Artifact ${ART_URL} not found (HTTP $HTTP). Aborting."
                exit 2
              fi
              echo "Artifact exists. Proceeding to deploy..."
              '''
          }
        }
      }
    }

    stage('Deploy via Ansible to Staging') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'jenkins-ssh-key-id', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            ART_URL="${NEXUS_URL}/${NEXUS_REPO}/$VERSION_NUMBER/"
            ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i "${INVENTORY}" "${ANSIBLE_PLAYBOOK}" \
              --private-key "$SSH_KEY" -u "$SSH_USER" \
              -e "artifact_version=$VERSION_NUMBER" \
              -e "artifact_url=$ART_URL" \
              -e "target_group=staging"
            '''
        }
      }
    }

    stage('Manual Promotion') {
      steps {
        input message: 'Promote to production?', ok: 'Promote'
      }
    }

    stage('Deploy via Ansible to Production') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'jenkins-ssh-key-id', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            ART_URL="${NEXUS_URL}/${NEXUS_REPO}/$VERSION_NUMBER/"
            ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i "${INVENTORY}" "${ANSIBLE_PLAYBOOK}" \
              --private-key "$SSH_KEY" -u "$SSH_USER" \
              -e "artifact_version=$VERSION_NUMBER" \
              -e "artifact_url=$ART_URL" \
              -e "target_group=production"
            '''
        }
      }
    }

  }
}













// pipeline {
//   agent { label 'ansible-agent' }

//   parameters {
//     string(name: 'ARTIFACT_VERSION', defaultValue: '1.0.0', description: 'Artifact version to deploy')
//     choice(name: 'TARGET_ENV', choices: ['staging','production'], description: 'Target environment for deploy')
//   }

//   environment {
//     NEXUS_HOST = '3.124.184.40'                         // e.g. 3.75.207.59
//     NEXUS_REPO = 'nexus/content/repositories/releases/php-crud-app'     // adjust to your repo path
//     NEXUS_BASE_URL = "http://${NEXUS_HOST}:8081"
//     ARTIFACT_FILE = "phpapp-${ARTIFACT_VERSION}.zip"
//     ARTIFACT_PATH = "artifacts/${ARTIFACT_FILE}"
//     NEXUS_UPLOAD_URL = "${NEXUS_BASE_URL}/${NEXUS_REPO}/${ARTIFACT_VERSION}/${ARTIFACT_FILE}"
//     NEXUS_DOWNLOAD_URL = "${NEXUS_BASE_URL}/${NEXUS_REPO}/${ARTIFACT_VERSION}/${ARTIFACT_FILE}"
//     INVENTORY = "hosts.ini"
//     ANSIBLE_PLAYBOOK = "deploy-app.yml"
//   }

//   stages {
//     stage('Checkout') {
//       steps { checkout scm }
//     }

//     stage('Package') {
//       steps {
//         sh '''#!/bin/bash
//           set -euo pipefail
//           # ensure the artifacts folder exists
//           mkdir -p artifacts
//           # create a zip of the contents of deploy_app/files (flat)
//           pushd deploy_app/files
//           zip -r "${WORKSPACE}/${ARTIFACT_PATH}" .  # this zips the files themselves, not the top folder
//           popd
//           ls -lh artifacts || true
//         '''
//       }
//       post { success { archiveArtifacts artifacts: 'artifacts/*.zip', allowEmptyArchive: false } }
//     }

//     stage('Upload to Nexus') {
//       steps {
//         withCredentials([usernamePassword(credentialsId: 'nexus-user-id', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PSW')]) {
//           sh '''#!/usr/bin/env bash
// set -euo pipefail
// echo "Uploading ${ARTIFACT_PATH} to ${NEXUS_UPLOAD_URL}"
// if [[ ! -f "${WORKSPACE}/${ARTIFACT_PATH}" ]]; then
//   echo "ERROR: artifact not found: ${WORKSPACE}/${ARTIFACT_PATH}"
//   exit 2
// fi
// curl --fail -S -u "${NEXUS_USER}:${NEXUS_PSW}" -T "${WORKSPACE}/${ARTIFACT_PATH}" "${NEXUS_UPLOAD_URL}"
// echo "Upload completed"
// '''
//         }
//       }
//     }

//     stage('Deploy via Ansible') {
//       steps {
//         withCredentials([sshUserPrivateKey(credentialsId: 'jenkins-ssh-key-id', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
//           sh '''#!/usr/bin/env bash
// set -euo pipefail
// ANSIBLE_HOST_KEY_CHECKING=False
// # pass artifact info to Ansible: artifact_version and artifact_url
// ansible-playbook -i "${INVENTORY}" "${ANSIBLE_PLAYBOOK}" \
//   --private-key "$SSH_KEY" -u "$SSH_USER" \
//   -e "artifact_version=${ARTIFACT_VERSION}" \
//   -e "artifact_url=${NEXUS_DOWNLOAD_URL}" \
//   -e "target_env=${TARGET_ENV}"
// '''
//         }
//       }
//     }

//     stage('Smoke Test') {
//       steps {
//         // adjust staging host or use DNS
//         sh '''#!/usr/bin/env bash
// set -euo pipefail
// URL="http://63.176.130.243/"   # change to your app URL
// STATUS=$(curl -s -o /dev/null -w '%{http_code}' "$URL" || echo "000")
// echo "Health check HTTP code: $STATUS"
// if [[ "$STATUS" != "200" ]]; then
//   echo "Smoke test failed: $STATUS"
//   exit 1
// fi
// '''
//       }
//     }
//   }

//   post {
//     success { echo "Pipeline completed. Artifact version: ${params.ARTIFACT_VERSION}" }
//     failure { echo "Pipeline FAILED for version ${params.ARTIFACT_VERSION}" }
//   }
// }
