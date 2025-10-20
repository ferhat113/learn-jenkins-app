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
                stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'ubuntu:22.04'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            apt-get update && apt-get install -y \
                                curl \
                                && curl -sSL https://deb.nodesource.com/setup_18.x | bash - \
                                && apt-get install -y nodejs \
                                && npm install -g playwright \
                                && npm install serve \
                                && serve -s build & \
                                sleep 10 \
                                && npx playwright test --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Local E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
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
                    npm install netlify-cli@17.37.0 node-jq
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
                    NETLIFY_AUTH_TOKEN=$NETLIFY_AUTH_TOKEN node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    NETLIFY_AUTH_TOKEN=$NETLIFY_AUTH_TOKEN node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                '''
            }
        }
        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }
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