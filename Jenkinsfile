pipeline {
    agent { label 'docker-agent' } // Forces the job to run on the Slave node
    stages {
        stage('Test Docker') {
            steps {
                sh 'docker --version'
                sh 'docker run hello-world'
            }
        }
    }
}
