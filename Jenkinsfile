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
        # Create virtual environment
        python3 -m venv venv

        # Activate virtual environment
        source venv/bin/activate

        # Upgrade pip inside venv
        python -m pip install --upgrade pip

        # Install requirements inside venv
        python -m pip install -r requirements.txt
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
