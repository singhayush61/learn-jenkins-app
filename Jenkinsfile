pipeline {
    agent any

    stages {
        stage('Run Tests'){
            parallel{
                stage('Test'){
                agent{
                    docker{
                        image 'node:18-alpine'
                        reuseNode true
                    }
                }
                steps{
                    sh '''
                    echo "hello"
                    '''
                }
            }

                stage('E2E'){
                    agent{
                        docker{
                        i   mage 'mcr.microsoft.com/playwright:v1.58.2-noble'
                            reuseNode true
                        }
                    }
                    steps{
                        sh '''
                        npm install serve
                        node_modules/.bin/serve -s build &
                        sleep 10
                        npx playwright test
                        '''
                    }
                }
            }
    }
        }
            
    
    post{
        always{
            junit 'test-results/junit.xml'
        }
    }
}
