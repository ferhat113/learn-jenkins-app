pipeline {
    // Agent definition for the entire pipeline
    agent any

    // Define the image name globally. 
    environment {
        // Jenkins is on the Worker node (192.168.0.33), so this is the registry host.
        IMAGE_HOST = "192.168.0.33"
        IMAGE_NAME = "${IMAGE_HOST}:5000/my-react-app:${env.BUILD_NUMBER}"
        LOCAL_PORT = "8080"  // Port for local deployment (on 192.168.0.33)
        REMOTE_PORT = "8081" // Port for remote deployment (on 192.168.0.34)
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
            steps {
                // FIX 1: Create Dockerfile outside of withDockerContainer context to ensure shell stability
                sh '''
                    echo "FROM nginx:alpine" > Dockerfile
                    echo "COPY build /usr/share/nginx/html" >> Dockerfile
                    echo "EXPOSE 80" >> Dockerfile
                '''

                // Now run the Docker client container to build and push the image
                script {
                    docker.image('docker:24.0-cli').withRun('-v /var/run/docker.sock:/var/run/docker.sock -u 0') { container ->
                        sh '''
                            echo "--- Building Docker Image: ${IMAGE_NAME} ---"
                            
                            # Build the image 
                            docker build -t ${IMAGE_NAME} .
                            
                            # Push the image to the local registry (192.168.0.33:5000)
                            docker push ${IMAGE_NAME}
                            echo "--- Docker Image Pushed to Registry: ${IMAGE_NAME} ---"
                        '''
                    }
                }
            }
        }

        stage('Deploy Locally (Worker Node - 192.168.0.33)') {
            // Deploying on the Jenkins host, so we use the Docker client directly.
            agent {
                docker {
                    image 'docker:24.0-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock -u 0' 
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "--- Starting local Docker deployment to ${IMAGE_HOST}:${LOCAL_PORT} ---"
                    
                    CONTAINER_NAME="react-app-local"

                    # Stop and remove the old container
                    docker ps -a --format '{{.Names}}' | grep ${CONTAINER_NAME} && docker rm -f ${CONTAINER_NAME} || true
                    
                    # The image is already local, just run it.
                    docker run -d --name ${CONTAINER_NAME} -p ${LOCAL_PORT}:80 ${IMAGE_NAME}
                    
                    echo "Local Deployment successful! App is running on ${IMAGE_HOST}:${LOCAL_PORT}"
                '''
            }
        }

        stage('Deploy Remotely (Manager Node - 192.168.0.34)') {
            agent {
                docker {
                    image 'instrumentisto/rsync-ssh' 
                    args '-u 0' // Run as root to avoid UID issues
                }
            }
            steps {
                // FIX 2: Use only usernameVariable, the key file is handled entirely by sshagent.
                withCredentials([sshUserPrivateKey(credentialsId: 'SSH_SERVER_CREDENTIALS', usernameVariable: 'SSH_USER')]) {
                    // CRITICAL FIX: sshagent must wrap the sh block to inject the credentials socket.
                    sshagent(['SSH_SERVER_CREDENTIALS']) {
                        sh '''
                            echo "--- Starting remote Docker deployment to 192.168.0.34:${REMOTE_PORT} ---"
                            
                            # --- Configuration Variables ---
                            REMOTE_HOST="192.168.0.34"
                            CONTAINER_NAME="react-app-remote"
                            
                            SSH_TARGET="${SSH_USER}@${REMOTE_HOST}"
                            SSH_OPTS="-o StrictHostKeyChecking=no" # No need for -i, sshagent handles it.

                            # 1. Stop and remove the old container on the remote host (192.168.0.34)
                            ssh ${SSH_OPTS} ${SSH_TARGET} "docker ps -a --format '{{.Names}}' | grep ${CONTAINER_NAME} && docker rm -f ${CONTAINER_NAME}" || true
                            
                            # 2. Pull the latest image from the worker node's registry (192.168.0.33:5000)
                            ssh ${SSH_OPTS} ${SSH_TARGET} "docker pull ${IMAGE_NAME}"

                            # 3. Run the new container, mapping remote port 8081 to container port 80
                            ssh ${SSH_OPTS} ${SSH_TARGET} "docker run -d --name ${CONTAINER_NAME} -p ${REMOTE_PORT}:80 ${IMAGE_NAME}"
                            
                            echo "Remote Deployment successful! App is running on ${REMOTE_HOST}:${REMOTE_PORT}"
                        '''
                    }
                }
            }
        }
    }
}
