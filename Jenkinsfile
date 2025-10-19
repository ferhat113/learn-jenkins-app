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
            
            // Use the node:18-alpine Docker image for running unit tests
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            
            // CORRECT SYNTAX: Use the 'environment' block to set variables for the whole stage.
            environment {
                JEST_JUNIT_OUTPUT_DIR = 'jest-results'
                JEST_JUNIT_OUTPUT_NAME = 'junit.xml'
            }

            steps {
                sh '''
                    echo "--- Running Unit Tests and configuring JUnit report output ---"
                    # Ensure the output directory exists
                    mkdir -p jest-results
                    
                    # Run tests. The environment variables set above direct the 'jest-junit' reporter.
                    npm test
                '''
            }
            
            // Post-build actions to always execute
            post {
                // Publish the JUnit test results
                always {
                    // This path now correctly matches the environment variable configuration in the 'environment' block.
                    junit "${JEST_JUNIT_OUTPUT_DIR}/${JEST_JUNIT_OUTPUT_NAME}"
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
