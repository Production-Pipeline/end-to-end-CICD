pipeline {
    agent any
    tools {
        nodejs 'nodejs-20'
    }
    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_USERNAME = credentials('mongo-username')
        MONGO_PASSWORD = credentials('mongo-password')
        SONAR_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Installing Dependencies') {
            steps {
                sh 'npm install --no-audit'
            }
        }
        stage('dependencies scanning'){
            parallel{
                stage('audit'){
                    steps{
                        sh 'npm audit --audit-level=critical || true'
                    }
                }
                stage('owasp'){
                    steps{
                        dependencyCheck additionalArguments: '''
                            --scan "./"
                            --out "./"
                            --format "ALL"
                            --prettyPrint''', odcInstallation: 'dep-check-10'
                        dependencyCheckPublisher failedTotalCritical: 6, pattern: 'dependency-check-report.xml', stopBuild: true
                         
                    }
                }

            }
        }
        stage('unit testing'){
            steps{
                sh 'npm test'
            }
        }
        stage('code coverage'){
            steps{
                    catchError(buildResult: 'SUCCESS', message: 'holy shit!!!', stageResult: 'UNSTABLE') {
                        sh 'npm run coverage'
                }
            }
        }
        stage('SAST'){
            steps{
                timeout(time: 60, unit: 'SECONDS') {
                    withSonarQubeEnv('sonar-server') {
                        sh '''
                            $SONAR_HOME/bin/sonar-scanner \
                                -Dsonar.projectKey=production-pipeline \
                                -Dsonar.sources=app.js \
                                -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info \
                        '''
                    }
                }
                waitForQualityGate abortPipeline: true
            }
        }
        stage('docker build'){
            steps{
                sh 'docker build -t sunilpolaki/production:$GIT_COMMIT .'
            }
        }
        stage('trivy'){
            steps{
                sh '''
                    trivy image sunilpolaki/production:$GIT_COMMIT \
                        --severity LOW,MEDIUM,HIGH \
                        --exit-code 0 \
                        --quiet \
                        --format json -o image-medium-results.json 

                    trivy image sunilpolaki/production:$GIT_COMMIT \
                        --severity CRITICAL \
                        --exit-code 0 \
                        --quiet \
                        --format json -o image-critical-results.json
                '''
            }
            post {
                always {
                    sh '''
                        trivy convert --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output image-medium-results.html image-medium-results.json

                        trivy convert --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output image-critical-results.html image-critical-results.json

                        # Convert to JUnit XML reports
                        trivy convert --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output image-medium-results.xml image-medium-results.json

                        trivy convert --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output image-critical-results.xml image-critical-results.json
                    '''

                    junit allowEmptyResults: true, testResults: 'image-*.xml'
                }
            }
        }
        stage('docker push'){
            steps{
                withDockerRegistry(credentialsId: 'docker-cred', url: "") {
                    sh 'docker push sunilpolaki/production:$GIT_COMMIT'
                }
            }
        }
        stage('aws'){
            when {
                branch comparator: 'REGEXP', pattern: 'feature.*'
            }
            steps{
                script{
                    sshagent(['ssh']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@13.201.225.128 '
                            if sudo docker ps -a | grep -q "solar-system"; then
                                echo "Container found. Stopping..."
                                sudo docker stop solar-system && sudo docker rm solar-system
                                echo "Container stopped and removed."
                            fi

                            echo "Running new container..."
                            sudo docker run --name solar-system \\
                                -e MONGO_URI=${env.MONGO_URI} \\
                                -e MONGO_USERNAME=${env.MONGO_USERNAME} \\
                                -e MONGO_PASSWORD=${env.MONGO_PASSWORD} \\
                                -p 3000:3000 -d sunilpolaki/production:$GIT_COMMIT
                        '
                        """
                    }

                }
            }
        }
        stage("integration testing"){
            when {
                branch comparator: 'REGEXP', pattern: 'feature.*'
            }
            steps{
                withAWS(credentials: 'aws-aws-iam-s3', region: 'ap-south-1') {
                    sh '''
                        bash integration.sh
                    '''
                }

            }
            
        }
    }
    post {
        always {
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-report.html', reportName: 'dpckeck HTML Report', reportTitles: '', useWrapperFileDirectly: true])
            junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml'
            junit allowEmptyResults: true, testResults: 'test-results.xml'
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'image-medium-results.html', reportName: 'trivy medium report', reportTitles: '', useWrapperFileDirectly: true])
            junit allowEmptyResults: true, testResults: 'image-medium-results.xml'
            junit allowEmptyResults: true, testResults: 'image-critical-results.xml'
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'image-critical-results.html', reportName: 'trivy critical Report', reportTitles: '', useWrapperFileDirectly: true])

    
        }
    }
}     