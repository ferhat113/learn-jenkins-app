pipeline {
    // Agent definition for the entire pipeline
    agent any

    // Define the image name globally, using the build number for uniqueness
    environment {
        IMAGE_NAME = "my-react-app:${env.BUILD_NUMBER}"
    }

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
        
        stage('Package (Docker Image)') {
            // Run on the same agent that has the workspace and build files.
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "--- Building Docker Image: ${IMAGE_NAME} ---"
                    
                    # Create a simple Nginx Dockerfile in the workspace
                    cat > Dockerfile <<EOF
                    FROM nginx:alpine
                    # Copy the static assets from the React build folder
                    COPY build /usr/share/nginx/html
                    EXPOSE 80
                    EOF

                    # Build the image and tag it
                    docker build -t ${IMAGE_NAME} .
                    
                    # Push the image to the local registry (required for the worker node to pull it)
                    docker push ${IMAGE_NAME}
                    echo "--- Docker Image Pushed ---"
                '''
            }
        }

        stage('Deploy to Server') {
            // Use a dedicated tool image that already has the SSH client installed.
            agent {
                docker {
                    image 'instrumentisto/rsync-ssh' 
                    // Run as root (UID 0) to avoid the "No user exists for uid 1000" error
                    args '-u 0' 
                }
            }
            steps {
                // Expose the key file path (SSH_KEY) and username (SSH_USER).
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'SSH_SERVER_CREDENTIALS', 
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        echo "--- Starting remote Docker deployment to 192.168.0.13 ---"
                        
                        # --- Configuration Variables ---
                        REMOTE_HOST="192.168.0.13"
                        REMOTE_PORT="8080"
                        CONTAINER_NAME="react-app-production"
                        
                        # FIX 1: Enclose $SSH_KEY in quotes to fix the space-in-path error
                        chmod 600 "$SSH_KEY"
                        
                        # Define the SSH prefix command with quoted variables
                        # The -i "$SSH_KEY" ensures the key is passed correctly, despite spaces in the path.
                        SSH_COMMAND_PREFIX="ssh -i \"$SSH_KEY\" -o StrictHostKeyChecking=no $SSH_USER@$REMOTE_HOST"

                        # 1. Stop and remove the old container on the remote host
                        # || true prevents the pipeline from failing if the container doesn't exist
                        $SSH_COMMAND_PREFIX "docker ps -a --format '{{.Names}}' | grep ${CONTAINER_NAME} && docker rm -f ${CONTAINER_NAME}" || true
                        
                        # 2. Pull the latest image from the manager node's registry
                        $SSH_COMMAND_PREFIX "docker pull ${IMAGE_NAME}"

                        # 3. Run the new container, mapping host port ${REMOTE_PORT} to container port 80
                        $SSH_COMMAND_PREFIX "docker run -d --name ${CONTAINER_NAME} -p ${REMOTE_PORT}:80 ${IMAGE_NAME}"
                        
                        echo "Deployment successful! App is running on $REMOTE_HOST:$REMOTE_PORT"
                    '''
                }
            }
        }
    }
}
