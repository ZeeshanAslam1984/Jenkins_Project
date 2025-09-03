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
                sh "python3 -m venv ${VENV_DIR}"
                // Use venv/bin/pip directly, no source needed
                sh """
                    ${VENV_DIR}/bin/pip install --upgrade pip
                    ${VENV_DIR}/bin/pip install -r requirements.txt
                """
            }
        }

        stage('Test') {
            steps {
                // Run tests using the virtual environment Python directly
                sh """
                    ${VENV_DIR}/bin/pytest --maxfail=1 --disable-warnings -v
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
