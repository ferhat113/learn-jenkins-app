pipeline {
    // Agent definition for the entire pipeline
    agent any

    // Define the image name and host globally.
    environment {
        // Using 'localhost' as the registry host for a local deployment setup.
        IMAGE_HOST = "localhost"
        IMAGE_NAME = "${IMAGE_HOST}:5000/my-react-app:${env.BUILD_NUMBER}"
        LOCAL_PORT = "8080"  // Port for local deployment
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
        
        stage('Setup Registry') {
            // Use the Docker client container to manage the host's Docker daemon.
            agent {
                docker {
                    image 'docker:24.0-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock -u 0'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "--- Ensuring Docker Registry is Running on ${IMAGE_HOST}:5000 ---"
                    # 1. Stop and remove any old registry container named 'registry-app'
                    docker ps -a --filter "name=registry-app" --format "{{.ID}}" | xargs -r docker rm -f
                    
                    # 2. Run the new registry container on the host, bound to port 5000.
                    docker run -d -p 5000:5000 --name registry-app registry:2
                    echo "--- Registry is now ready ---"
                '''
            }
        }

        stage('Package (Docker Image)') {
            steps {
                // Create Dockerfile outside of withDockerContainer context for stability
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
                            
                            # Push the image to the local registry (localhost:5000)
                            # This will now succeed because the registry-app container is running.
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
                    
                    # Run the new container, mapping host port 8080 to container port 80.
                    # It pulls from localhost:5000.
                    docker run -d --name ${CONTAINER_NAME} -p ${LOCAL_PORT}:80 ${IMAGE_NAME}
                    
                    echo "Local Deployment successful! App is running on ${IMAGE_HOST}:${LOCAL_PORT}"
                '''
            }
        }
    }
}
