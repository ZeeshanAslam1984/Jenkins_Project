pipeline {
    agent any

    environment {
        VENV_DIR = "venv"
    }

    stages {
        stage('lint and format') {
            parallel {
                stage('linting') {
                    steps {
                       bat "ping -n 31 127.0.0.1 >nul"

                    }
                }
                stage('formatting') {
                    steps {
                       bat "ping -n 31 127.0.0.1 >nul"

                    }
                }
            }
        }

        stage('Setup') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'server-creds', usernameVariable: "myuser", passwordVariable: "mypassword")]) {
                    bat '''
                        echo %myuser%
                        echo %mypassword%
                    '''
                }

                bat "python -m venv %VENV_DIR%"

                bat """
                    %VENV_DIR%\\Scripts\\python -m pip install --upgrade pip
                    %VENV_DIR%\\Scripts\\python -m pip install --requirement requirements.txt --cache-dir=%WORKSPACE%\\.pip-cache
                """
            }
        }

        stage('Test') {
            steps {
                bat "%VENV_DIR%\\Scripts\\pytest --maxfail=1 --disable-warnings -v"
            }
        }
    }
}
