pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '89df36c3-02cd-4dcc-a7af-8f6abd236972'
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
            environment{
                AWS_S3_BUCKET = 'learn-jenkins-202602'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                sh '''
                    aws --version
                    echo "Hello s3!" > index.html
                    aws s3 cp index.html s3://$AWS_S3_BUCKET/index.html
                    aws s3 ls
                '''
                }
                
            }
        }

        stage('Docker Cleanup') {
            agent any // This runs on the host server where Docker is installed
            steps {
                sh '''
                    echo "--- Checking Disk Space ---"
                    df -h
                    
                    echo "--- Checking Inodes ---"
                    df -i
                    
                    echo "--- Pruning Docker ---"
                    
                    docker system prune -f
                    '''
                }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                    args '-u root'
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

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                            args '-u root:root'
                        }
                    }

                    steps {
                        sh '''
                            #test -f build/index.html
                            npm test
                            rm -rf test-results
                            mkdir test-results
                            JEST_JUNIT_OUTPUT_DIR="./test-results/" \
                            JEST_JUNIT_OUTPUT_NAME="junit.xml" \
                            chmod 777 test-results
                            npm test -- --watchAll=false --forceExit
                            
                            '''
                    }
                    post {
                        always {
                            sh 'chmod -R 777 test-results'
                            junit 'test-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            args '-u root'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npx serve -s build -l 3000 &
                            sleep 10
                            npx playwright test  --reporter=html
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

        stage('Deploy staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://simple-project-ayush.netlify.app/'
            }

            steps {
                sh '''
                    netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > deploy-output.json
                    CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)
                    npx playwright test  --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://simple-project-ayush.netlify.app/'
            }

            steps {
                sh '''
                    node --version
                    netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod
                    npx playwright test  --reporter=html
                '''
            }

            post {
                always {
                    sh 'chown -R 1000:1000 .'
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }

}