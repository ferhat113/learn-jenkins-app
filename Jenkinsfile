pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker 'node:18-alpine'
                reuseNode true
            }
        }
            steps {
                sh '''
                echo "Hello World !!!"
                ls -la
                node --version
                npm --version
                npm ci
                npm run build
                ls -la
                '''
            }
        }
    }
