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
                            npm test
                            # 1. Clean and create the folder
                            rm -rf test-results
                            mkdir -p test-results
                            chmod 777 test-results
                            
                            # 2. Set the destination for the XML file
                            export JEST_JUNIT_OUTPUT_DIR="./test-results"
                            export JEST_JUNIT_OUTPUT_NAME="junit.xml"
                            
                            # 3. Run the test script defined in package.json
                            # Use --watchAll=false to ensure Jest exits
                            # Use --forceExit to handle those 'open handles' mentioned in your logs
                            npm test -- --watchAll=false --forceExit
                            
                            '''
                    }
                    post {
                        always {
                            junit 'test-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.41.0-jammy'
                            args '-u root'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npx playwright install-deps
                            npx serve -s build -l 3000 &
                            sleep 10
                            npx playwright test  
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
                    image 'mcr.microsoft.com/playwright:v1.41.0-jammy'
                    image 'node:20.18'
                    reuseNode true
                    args '-u root'
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://simple-project-ayush.netlify.app/'
            }

            steps {
                sh '''
                    
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    npx netlify status
                    #npx netlify deploy --dir=build --json > deploy-output.json

                    # 1. Run deploy and SAVE the output to a file
                    npx netlify deploy --dir=build --json > deploy-output.json
                    
                    # 2. Extract the URL using the Node.js trick (No jq needed!)
                    CI_ENVIRONMENT_URL=$(node -p "require('./deploy-output.json').deploy_url")
                    
                    echo "Successfully deployed to: $CI_ENVIRONMENT_URL"
                    
                    #CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)
                    npx playwright install
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
                    image 'mcr.microsoft.com/playwright:v1.41.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://simple-project-ayush.netlify.app/'
            }

            steps {
                sh '''
                    node --version
                    npx netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    npx netlify status
                    npx netlify deploy --dir=build --prod
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