pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'cf6f0666-741a-4c5f-a6bc-c551ee146206'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
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

        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine' // Consider switching to a non-Alpine image if issues persist
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    echo "Current directory: $(pwd)"
                    ls -la

                    # Run the build command to double check its success
                    echo "Running npm run build to check for errors..."
                    npm run build
                    
                    # Deploy only if build is successful
                    echo "Deploying to Netlify..."
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }
    }
}
