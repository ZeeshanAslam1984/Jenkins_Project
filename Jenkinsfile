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
    steps {
        sh '''
        if [ -f requirements.txt ]; then
            pip3 install -r requirements.txt
        else
            echo "requirements.txt not found!"
        fi
        '''
    }
}


        stage('Test') {
            steps {
                sh "pytest"
            }
        }
    }    
}
