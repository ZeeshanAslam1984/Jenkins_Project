pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/ZeeshanAslam1984/Jenkins_Project.git', branch: 'main'
                sh "ls -ltr"
            }
        }
        stage('Setup') {
    sh 'pip install -r requirements.txt'
}


        stage('Test') {
            steps {
                sh "pytest"
            }
        }
    }    
}
