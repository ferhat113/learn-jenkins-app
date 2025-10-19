pipeline {
    // Agent definition for the entire pipeline
    agent any

    stages {

        stage('Build') {
            // Use the node:18-alpine Docker image for building
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "--- Starting Build Stage ---"
                    npm ci
                    npm run build
                    echo "--- Build completed. The 'build/' directory is ready. ---"
                '''
            }
        }

        stage('Tests') {
            
            // Use the node:18-alpine Docker image for running unit tests
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            
            // Set environment variables to configure the jest-junit reporter output path
            environment {
                JEST_JUNIT_OUTPUT_DIR = 'jest-results'
                JEST_JUNIT_OUTPUT_NAME = 'junit.xml'
            }

            steps {
                sh '''
                    echo "--- Running Unit Tests ---"
                    mkdir -p jest-results
                    npm test
                '''
            }
            
            post {
                always {
                    junit "${JEST_JUNIT_OUTPUT_DIR}/${JEST_JUNIT_OUTPUT_NAME}"
                }
            }
        }

        stage('Deploy to Server') {
            // Reusing node:18-alpine, but we must install the SSH client inside it.
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                // 1. Install SSH/SCP utility (essential for Alpine-based images)
                sh 'apk add --no-cache openssh-client'
                
                // 2. Use withCredentials and sshagent for secure deployment
                withCredentials([sshUserPrivateKey(credentialsId: 'SSH_SERVER_CREDENTIALS', keyFileVariable: 'SSH_KEY')]) {
                    sshagent(['SSH_SERVER_CREDENTIALS']) {
                        sh '''
                            echo "--- Starting secure deployment to 192.168.0.13 ---"
                            
                            # --- Configuration Variables ---
                            # IMPORTANT: Replace these placeholders with your actual server details
                            REMOTE_HOST="192.168.0.13"
                            REMOTE_USER="jenkins_deploy_user" 
                            REMOTE_PATH="/var/www/my-app" 
                            
                            # Optional: Ensure the remote directory exists before copying
                            # This uses 'ssh' to execute a remote command.
                            ssh -o StrictHostKeyChecking=no $REMOTE_USER@$REMOTE_HOST "mkdir -p $REMOTE_PATH"

                            # Copy the contents of the 'build' directory recursively to the remote server
                            # The 'build/.' ensures we copy the *contents* of 'build', not the 'build' folder itself.
                            scp -r -o StrictHostKeyChecking=no build/. $REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH
                            
                            echo "Deployment successful to $REMOTE_HOST:$REMOTE_PATH"
                        '''
                    }
                }
            }
        }
    }
}
