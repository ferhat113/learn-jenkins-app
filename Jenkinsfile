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
                    echo "--- Running npm installation and build ---"
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    echo "--- Verifying build directory ---"
                    ls -la build
                '''
            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "--- Installing Netlify CLI for deployment ---"
                    npm install netlify-cli@17.37.0
                    node_modules/.bin/netlify --version

                    echo "--- Creating .netlify/state.json manually ---"
                    mkdir -p .netlify
                    echo '{ "siteId": "'"$NETLIFY_SITE_ID"'" }' > .netlify/state.json
                    cat .netlify/state.json

                    echo "--- Checking build directory ---"
                    ls -la build

                    echo "--- Checking Netlify status ---"
                    NETLIFY_AUTH_TOKEN=$NETLIFY_AUTH_TOKEN node_modules/.bin/netlify status

                    echo "--- Deploying to production ---"
                    NETLIFY_AUTH_TOKEN=$NETLIFY_AUTH_TOKEN node_modules/.bin/netlify deploy --dir=build
                '''
            }
        }

        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "--- Installing Netlify CLI for deployment ---"
                    npm install netlify-cli@17.37.0
                    node_modules/.bin/netlify --version

                    echo "--- Creating .netlify/state.json manually ---"
                    mkdir -p .netlify
                    echo '{ "siteId": "'"$NETLIFY_SITE_ID"'" }' > .netlify/state.json
                    cat .netlify/state.json

                    echo "--- Checking build directory ---"
                    ls -la build

                    echo "--- Checking Netlify status ---"
                    NETLIFY_AUTH_TOKEN=$NETLIFY_AUTH_TOKEN node_modules/.bin/netlify status

                    echo "--- Deploying to production ---"
                    NETLIFY_AUTH_TOKEN=$NETLIFY_AUTH_TOKEN node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }
    }
}