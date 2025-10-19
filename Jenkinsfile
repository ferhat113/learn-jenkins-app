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
            // FIX: We switch the base image to `debian:stable-slim`.
            // This is a minimal, Debian-based image that typically allows for easier package installation 
            // of tools like SSH than the Alpine images in a non-root environment.
            // NOTE: `reuseNode true` is removed because we are using a different base image.
            agent {
                docker {
                    image 'debian:stable-slim' 
                }
            }
            steps {
                // 1. Install SSH/SCP utility using `apt-get` on the Debian-based image.
                // We assume the Jenkins user has permission to install packages within this temporary container.
                sh 'apt-get update -y && apt-get install openssh-client -y'
                
                // 2. Use withCredentials and sshagent for secure deployment.
                withCredentials([sshUserPrivateKey(credentialsId: 'SSH_SERVER_CREDENTIALS', keyFileVariable: 'SSH_KEY')]) {
                    sshagent(['SSH_SERVER_CREDENTIALS']) {
                        sh '''
                            echo "--- Starting secure deployment to 192.168.0.13 ---"
                            
                            # --- Configuration Variables (Adjust these as needed) ---
                            REMOTE_HOST="192.168.0.13"
                            REMOTE_USER="jenkins_deploy_user" 
                            REMOTE_PATH="/var/www/my-app" 
                            
                            # Ensure the remote directory exists
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
