pipeline {
    // Agent definition for the entire pipeline (used if no stage-specific agent is defined)
    agent any

    stages {

        stage('Build') {
            // Use the node:18-alpine Docker image for building
            agent {
                docker {
                    image 'node:18-alpine'
                    // Reuse the container if possible for faster subsequent steps
                    reuseNode true
                }
            }
            steps {
                // Execute standard node/npm build commands
                sh '''
                    echo "--- Starting Build Stage ---"
                    node --version
                    npm --version
                    
                    # Install dependencies from package-lock.json (faster and safer than npm install)
                    npm ci
                    
                    # Run the build process (e.g., creating the 'build' directory)
                    npm run build
                    
                    echo "--- Build completed. Contents of workspace: ---"
                    ls -la
                '''
            }
        }

        stage('Tests') {
            // This stage now directly contains the test steps and post-actions.
            
            // Use the node:18-alpine Docker image for running unit tests
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    echo "--- Running Unit Tests ---"
                    # Run tests (assuming this command generates a JUnit XML report)
                    npm test
                '''
            }
            
            // Post-build actions to always execute
            post {
                // Publish the JUnit test results
                always {
                    junit 'jest-results/junit.xml'
                }
            }
        }

        stage('Deploy') {
            // Use the node:18-alpine Docker image for deployment tasks
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "--- Starting Deployment Setup ---"
                    # Install Netlify CLI globally or locally to run deployment commands
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    
                    # Add your actual deployment logic here, e.g.:
                    # node_modules/.bin/netlify deploy --dir=build --prod --auth $NETLIFY_AUTH_TOKEN
                '''
            }
        }
    }
}
