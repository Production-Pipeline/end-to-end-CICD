pipeline {
    agent any
    tools {
        nodejs 'nodejs-25'
    }
    stages {
        stage('Installing Dependencies') {
            steps {
                sh 'npm install --no-audit'
            }
        }
    }
}     