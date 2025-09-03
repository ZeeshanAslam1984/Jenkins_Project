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
                // Check if requirements.txt exists and install
                sh '''
                if [ -f requirements.txt ]; then
                    pip install -r requirements.txt
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
