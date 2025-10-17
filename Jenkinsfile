pipeline {
    agent any

    stages {
        /*

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        */
        //this is another comment

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            /*
            juste
            multiple 
            lines
            comment
            */

            steps {
                sh '''
                    #test -f build/index.html
                    npm test
                '''
            }
        }
    }

    post {
        //this is a comment
        always {
            junit 'test-results/junit.xml'
        }
    }
}
