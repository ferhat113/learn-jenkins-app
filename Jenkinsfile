pipeline {
    // Agent definition for the entire pipeline
    agent any

    // Define the image name globally, using the manager's IP for the registry
    environment {
        // Since Jenkins is on 192.168.0.14, we tag the image against this IP for the worker to pull.
        IMAGE_NAME = "192.168.0.14:5000/my-react-app:${env.BUILD_NUMBER}"
    }

    stages {

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'npm ci && npm run build'
            }
        }

        stage('Tests') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            environment {
                JEST_JUNIT_OUTPUT_DIR = 'jest-results'
                JEST_JUNIT_OUTPUT_NAME = 'junit.xml'
            }
            steps {
                sh 'mkdir -p jest-results && npm test'
            }
            post {
                always {
                    junit "${JEST_JUNIT_OUTPUT_DIR}/${JEST_JUNIT_OUTPUT_NAME}"
                }
            }
        }
        
        stage('Package (Docker Image)') {
            // FIX: Use the official Docker client image and mount the Docker socket
            agent {
                docker {
                    image 'docker:24.0-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                    reuseNode true // Use the same node where the build happened
                }
            }
            steps {
                sh '''
                    echo "--- Building Docker Image: ${IMAGE_NAME} ---"
                    
                    # Ensure Docker daemon is running (important in shared environments)
                    if ! docker info > /dev/null 2>&1; then
                        echo "Docker daemon is not running or socket is not accessible."
                        exit 1
                    fi
                    
                    # Create a simple Nginx Dockerfile in the workspace
                    cat > Dockerfile <<EOF
                    FROM nginx:alpine
                    COPY build /usr/share/nginx/html
                    EXPOSE 80
                    EOF

                    # Build the image and tag it against the manager's registry (192.168.0.14)
                    docker build -t ${IMAGE_NAME} .
                    
                    # Ensure the local registry is running on 192.168.0.14:5000 before pushing
                    docker push ${IMAGE_NAME}
                    echo "--- Docker Image Pushed to Registry: ${IMAGE_NAME} ---"
                '''
            }
        }

        stage('Deploy to Server') {
            agent {
                docker {
                    image 'instrumentisto/rsync-ssh' 
                    // FIX: Run as root (UID 0) to avoid the "No user exists for uid 1000" error
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
                        
                        # FIX 2: Use double quotes around $SSH_KEY to handle spaces in the path
                        chmod 600 "$SSH_KEY"
                        
                        # SSH prefix arguments
                        SSH_ARGS="-i \"$SSH_KEY\" -o StrictHostKeyChecking=no"

                        # 1. Stop and remove the old container on the remote host (192.168.0.13)
                        ssh ${SSH_ARGS} ${SSH_USER}@${REMOTE_HOST} "docker ps -a --format '{{.Names}}' | grep ${CONTAINER_NAME} && docker rm -f ${CONTAINER_NAME}" || true
                        
                        # 2. Pull the latest image from the manager node's registry (192.168.0.14)
                        ssh ${SSH_ARGS} ${SSH_USER}@${REMOTE_HOST} "docker pull ${IMAGE_NAME}"

                        # 3. Run the new container
                        ssh ${SSH_ARGS} ${SSH_USER}@${REMOTE_HOST} "docker run -d --name ${CONTAINER_NAME} -p ${REMOTE_PORT}:80 ${IMAGE_NAME}"
                        
                        echo "Deployment successful! App is running on $REMOTE_HOST:$REMOTE_PORT"
                    '''
                }
            }
        }
    }
}
