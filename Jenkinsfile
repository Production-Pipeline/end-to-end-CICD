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
        GIT_TOKEN = credentials('git-token')
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
        stage("update image tag"){
            when{
                branch 'PR*'
            }
            steps{
                sh 'git clone -b main https://github.com/Production-Pipeline/argo-cd.git'
                dir('argo-cd/kubernetes'){
                    sh '''
                        ##### Replace Docker Tag #####
                        git checkout main
                        git checkout -b feature-$BUILD_ID
                        sed -i "s#sunilp.*#sunilpolaki/production:$GIT_COMMIT#g" deployment.yml
                        cat deployment.yml


                        ##### Commit and Push to Feature Branch #####
                        git config --global user.email "chakrachandb@gmail.com"
                        git config --global user.name "chakribaggam456"
                        git remote set-url origin https://$GIT_TOKEN@github.com/Production-Pipeline/argo-cd.git
                        git add .
                        git commit -am "Updated docker image"
                        git push -u origin feature-$BUILD_ID
                    '''
                }

            }
        }
        stage('raising pr for main'){
            when {
              branch 'PR*'
            }
            steps {
                sh """
                curl -s -o response.json -w "%{http_code}" -X POST \\
                https://api.github.com/repos/Production-Pipeline/argo-cd/pulls \\
                -H "Accept: application/vnd.github+json" \\
                -H "Authorization: token ${GIT_TOKEN}" \\
                -H "Content-Type: application/json" \\
                -d '{
                    "title": "Updated Docker Image",
                    "body": "Updated docker image in deployment manifest",
                    "head": "feature-${BUILD_ID}",
                    "base": "main",
                    "assignees": ["chakribaggam456"]
                }'
                """
            }
        }
        stage("app deployed") {
            when{
                branch 'PR*'
            }
            steps {
                timeout(time: 1, unit: 'DAYS') {
                    input message: 'Is the new version of the app synced and deployed?', ok: 'Yes'
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            when {
                branch 'PR*'
            }
            steps {
                catchError(buildResult: 'SUCCESS', message: 'no worries', stageResult: 'UNSTABLE'){
                    sh '''
                    chmod 777 $(pwd)
                    docker run -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy zap-api-scan.py \
                    -t http://43.205.206.18:30000/api-docs/ \
                    -f openapi \
                    -r zap_report.html \
                    -w zap_report.md \
                    -J zap_json_report.json \
                    -x zap_xml_report.xml
                    -c zap_ignore_rules
                    '''
                }
            }
        }
    }
    post {
        always {
            script {
                if (fileExists('argo-cd')) {
                    sh 'rm -rf argo-cd'
                }
            }
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