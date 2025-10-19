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
            // FIX: Using a dedicated tool image that already has the SSH client installed.
            // This bypasses all permission issues related to `apt-get` or `apk add`.
            agent {
                docker {
                    image 'instrumentisto/rsync-ssh' 
                    // NOTE: `reuseNode true` is removed because we are using a different, dedicated image.
                }
            }
            steps {
                // IMPORTANT: The pipeline requires a Jenkins Credential with ID 'SSH_SERVER_CREDENTIALS'
                // configured as an "SSH Username with Private Key."
                withCredentials([sshUserPrivateKey(credentialsId: 'SSH_SERVER_CREDENTIALS', keyFileVariable: 'SSH_KEY')]) {
                    sshagent(['SSH_SERVER_CREDENTIALS']) {
                        sh '''
                            echo "--- Starting secure deployment to 192.168.0.13 ---"
                            
                            # --- Configuration Variables (Adjust these as needed) ---
                            REMOTE_HOST="192.168.0.13"
                            REMOTE_USER="worker1" # <<== REPLACE WITH YOUR ACTUAL DEPLOYMENT USER
                            REMOTE_PATH="/var/www/my-app"     # <<== REPLACE WITH YOUR ACTUAL REMOTE PATH
                            
                            # Ensure the remote directory exists
                            # The '-o StrictHostKeyChecking=no' is used to skip the first-time host key check, 
                            # necessary for automated pipelines.
                            ssh -o StrictHostKeyChecking=no $REMOTE_USER@$REMOTE_HOST "mkdir -p $REMOTE_PATH"

                            # Copy the contents of the 'build' directory recursively to the remote server
                            scp -r -o StrictHostKeyChecking=no build/. $REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH
                            
                            echo "Deployment successful to $REMOTE_HOST:$REMOTE_PATH"
                        '''
                    }
                }
            }
        }
    }
}
