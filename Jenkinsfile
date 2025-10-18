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
    }
}     