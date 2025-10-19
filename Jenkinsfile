pipeline {
    // Agent definition for the entire pipeline
    agent any

    // Define the image name globally. 
    environment {
        // Jenkins is on the Worker node (192.168.0.33), so this is the registry host.
        IMAGE_HOST = "192.168.0.33"
        IMAGE_NAME = "${IMAGE_HOST}:5000/my-react-app:${env.BUILD_NUMBER}"
        LOCAL_PORT = "8080"  // Port for local deployment (on 192.168.0.33)
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
        
        stage('Package (Docker Image)') {
            steps {
                // Create Dockerfile outside of withDockerContainer context to ensure shell stability
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
    }
}
