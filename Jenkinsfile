pipeline {
    agent any

    environment {
        IMAGE_NAME = "localhost:5000/my-react-app:${env.BUILD_NUMBER}"
        LOCAL_PORT = "8080"
        CONTAINER_NAME = "react-app-local"
    }

    stages {
        stage('Clone') {
            steps {
                // Replace with your repo URL
                git 'https://github.com/your-username/your-repo.git'
            }
        }

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
                sh '''
                    echo "FROM nginx:alpine" > Dockerfile
                    echo "COPY build /usr/share/nginx/html" >> Dockerfile
                    echo "EXPOSE 80" >> Dockerfile
                '''
                script {
                    docker.image('docker:24.0-cli').withRun('-v /var/run/docker.sock:/var/run/docker.sock -u 0') { 
                        sh '''
                            docker build -t ${IMAGE_NAME} .
                            docker push ${IMAGE_NAME}
                        '''
                    }
                }
            }
        }

        stage('Deploy Locally') {
            agent {
                docker {
                    image 'docker:24.0-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock -u 0'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    docker rm -f ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p ${LOCAL_PORT}:80 ${IMAGE_NAME}
                    echo "App running on localhost:${LOCAL_PORT}"
                '''
            }
        }
    }
}
