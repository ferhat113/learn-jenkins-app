pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    args '-v /tmp:/tmp'  // Optional: additional arguments for Docker container
                }
            }
            steps {
                script {
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
    }
}
