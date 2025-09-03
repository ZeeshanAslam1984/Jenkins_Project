pipeline {
    agent any
    parameters {
        string(name: 'ENVIRONMENT', defaultValue: 'dev', description: 'Specify the environment for deployment')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: "Run Tests in pipeline")
    }

    /*options {
        timeout(time: 1, unit: 'MINUTES')
    }*/
    environment {
        VENV_DIR = "venv"
    }

    stages {
        /*stage('lint and format') {
            parallel {
                stage('linting') {
                    steps {
                        sleep(time: 70, unit: 'SECONDS')
                    }
                }
                stage('formatting') {
                    steps {
                        sleep(time: 70, unit: 'SECONDS')
                    }
                }
            }
        }*/

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
            when {
                expression {
                    params.RUN_TESTS == true
                }
            }
            steps {
                bat "%VENV_DIR%\\Scripts\\pytest --maxfail=1 --disable-warnings -v"
                echo "testing application"
            }
        }

       stage('Deployment') { 
    steps {
        echo "deploying to ${params.ENVIRONMENT} environment"
        input message: "Do you want to proceed", ok: "Yes"
    }
}

    }
}
