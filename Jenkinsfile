pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'f1a2fa17-9354-49f2-aa47-9af81c17398b'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }
    stages {

        stage('AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args "--entrypoint=''"
                }
            }
            environment {
                AWS_S3_BUCKET = 'learn-jenkins-2026-05-16'
            }

            steps {
                withCredentials([usernamePassword(credentialsId: 'aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        echo 'hello s3' > index.html
                        aws s3 cp index.html s3://AWS_S3_BUCKET/index.html
                    '''
                }
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
                sh '''
                    echo 'Small Changes'
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        stage ('Test') {
            parallel {
                    stage('Unit Test') {
                        agent {
                            docker {
                                image 'node:18-alpine'
                                reuseNode true
                            }
                        }
                        steps {
                            sh '''
                                echo "Test Stage"
                                test -f build/index.html
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
                                image 'my-playwright'
                                reuseNode true
                            }
                        }
                        steps {
                            sh '''
                                serve -s build &
                                sleep 10
                                npx playwright test --reporter=html
                            '''
                        }
                        post {
                            always {
                                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local Report', reportTitles: '', useWrapperFileDirectly: true])
                                }
                            }
                    }

            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'STAGING_URL'
            }
            steps {
                sh '''
                    netlify --version
                    echo "Deploying to staging site id: $NETFLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > deploy-output.json
                    jq -r '.deploy_url' deploy-output.json
                    CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Staging E2E Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Deploy Prod') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://incandescent-croissant-f7853d.netlify.app'
            }

            steps {
                sh '''
                    node --version
                    npm install netlify-cli
                    netlify --version
                    echo "Deploying to production site id: $NETFLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Prod E2E Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }     
    }  
}
