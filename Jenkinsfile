pipeline {
    agent any
    tools {
        nodejs 'nodejs-20'
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
                        sh 'npm audit --audit-level=critical'
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
    }
}     