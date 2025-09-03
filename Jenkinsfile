pipeline {
    agent any

    environment {
        VENV_DIR = "venv"
        DB_HOST = '192.168.12.1'
        USERNAME = 'zeeshan'
        PASSWORD = 'zesh123'
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/ZeeshanAslam1984/Jenkins_Project.git', branch: 'main'
                bat 'dir'
            }
        }

        stage('Setup') {
            steps {
                // Create virtual environment
                bat "python -m venv %VENV_DIR%"
                
                // Upgrade pip and install requirements
                bat """
                    %VENV_DIR%\\Scripts\\python -m pip install --upgrade pip
                    %VENV_DIR%\\Scripts\\python -m pip install -r requirements.txt
                    echo The Database IP is: %DB_HOST%
                """
            }
        }

        stage('Test') {
            steps {
                // Run tests using virtual environment Python
                bat "%VENV_DIR%\\Scripts\\pytest --maxfail=1 --disable-warnings -v"
                echo "The DB username: ${env.USERNAME} and the password is: ${env.PASSWORD}"
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            bat "rmdir /s /q %VENV_DIR%"
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check errors above.'
        }
    }
}
