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
            // Using a dedicated tool image that already has the SSH client installed.
            agent {
                docker {
                    image 'instrumentisto/rsync-ssh' 
                    // FIX: Run as root (UID 0) to avoid the "No user exists for uid 1000" error
                    args '-u 0' 
                }
            }
            steps {
                // We use withCredentials to expose the key file path (SSH_KEY) and username (SSH_USER).
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'SSH_SERVER_CREDENTIALS', 
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        echo "--- Starting secure deployment to 192.168.0.13 ---"
                        
                        # --- Configuration Variables (Adjust these as needed) ---
                        REMOTE_HOST="192.168.0.13"
                        REMOTE_PATH="/var/www/my-app"
                        
                        # Set file permissions for the private key (MANDATORY for SSH authentication)
                        chmod 600 $SSH_KEY
                        
                        # Ensure the remote directory exists, using the private key for authentication
                        # -i $SSH_KEY is the identity file (private key)
                        # -o StrictHostKeyChecking=no skips the prompt for the host key
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@$REMOTE_HOST "mkdir -p $REMOTE_PATH"

                        # Copy the contents of the 'build' directory recursively to the remote server
                        scp -r -i $SSH_KEY -o StrictHostKeyChecking=no build/. $SSH_USER@$REMOTE_HOST:$REMOTE_PATH
                        
                        echo "Deployment successful to $REMOTE_HOST:$REMOTE_PATH"
                    '''
                }
            }
        }
    }
}
