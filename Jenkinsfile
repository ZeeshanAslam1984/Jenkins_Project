pipeline {
    agent any

    environment {
        VENV_DIR = "venv"
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/ZeeshanAslam1984/Jenkins_Project.git', branch: 'main'
                sh 'ls -ltr'
            }
        }

        stage('Setup') {
            steps {
                // Create a virtual environment
                sh "python3 -m venv ${VENV_DIR}"
                // Activate venv and install requirements
                sh """
                    source ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                """
            }
        }

        stage('Test') {
            steps {
                // Run tests inside virtual environment
                sh """
                    source ${VENV_DIR}/bin/activate
                    pytest --maxfail=1 --disable-warnings -v
                """
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            sh "rm -rf ${VENV_DIR}"
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check errors above.'
        }
    }
}
