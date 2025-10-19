pipeline {
    agent any

    environment {
        IMAGE_HOST = "192.168.0.33"
        IMAGE_NAME = "${IMAGE_HOST}:5000/my-react-app:${env.BUILD_NUMBER}"
        LOCAL_PORT = "8080"
        REMOTE_PORT = "8081"
        CONTAINER_NAME_LOCAL = "react-app-local"
        CONTAINER_NAME_REMOTE = "react-app-remote"
        REMOTE_HOST = "192.168.0.34"
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

        stage('Test') {
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

        stage('Create Dockerfile') {
            steps {
                writeFile file: 'Dockerfile', text: '''
                    FROM nginx:alpine
                    COPY build /usr/share/nginx/html
                    EXPOSE 80
                '''
            }
        }

        stage('Build and Push Docker Image') {
            agent {
                docker {
                    image 'docker:24.0-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock -u 0'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "--- Building Docker Image: ${IMAGE_NAME} ---"
                    docker build -t ${IMAGE_NAME} .
                    docker push ${IMAGE_NAME}
                    echo "--- Docker Image Pushed to Registry ---"
                '''
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
                    echo "--- Deploying Locally on ${IMAGE_HOST}:${LOCAL_PORT} ---"
                    docker rm -f ${CONTAINER_NAME_LOCAL} || true
                    docker run -d --name ${CONTAINER_NAME_LOCAL} -p ${LOCAL_PORT}:80 ${IMAGE_NAME}
                    echo "✅ Local deployment complete!"
                '''
            }
        }

        stage('Deploy Remotely') {
            agent {
                docker {
                    image 'instrumentisto/rsync-ssh'
                    args '-u 0'
                }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'SSH_SERVER_CREDENTIALS', usernameVariable: 'SSH_USER')]) {
                    sshagent(['SSH_SERVER_CREDENTIALS']) {
                        sh '''
                            echo "--- Deploying Remotely to ${REMOTE_HOST}:${REMOTE_PORT} ---"
                            SSH_TARGET="${SSH_USER}@${REMOTE_HOST}"
                            SSH_OPTS="-o StrictHostKeyChecking=no"

                            ssh ${SSH_OPTS} ${SSH_TARGET} "docker rm -f ${CONTAINER_NAME_REMOTE} || true"
                            ssh ${SSH_OPTS} ${SSH_TARGET} "docker pull ${IMAGE_NAME}"
                            ssh ${SSH_OPTS} ${SSH_TARGET} "docker run -d --name ${CONTAINER_NAME_REMOTE} -p ${REMOTE_PORT}:80 ${IMAGE_NAME}"
                            echo "✅ Remote deployment complete!"
                        '''
                    }
                }
            }
        }
    }
}
