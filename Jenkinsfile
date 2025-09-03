pipeline {
    agent any

    environment {
        VENV_DIR = "venv"
        DB_HOST = '192.168.12.1'
        USERNAME = 'zeeshan'
        PASSWORD = 'zesh123'
    }

    stages {

        stage('Setup') {
    steps {
        
        bat "python -m venv %VENV_DIR%"
                
        bat """
            %VENV_DIR%\\Scripts\\python -m pip install --upgrade pip
            %VENV_DIR%\\Scripts\\python -m pip install --requirement requirements.txt --cache-dir=%WORKSPACE%\\.pip-cache
            echo The Database IP is: %DB_HOST%
        """
    }
}

        stage('Test') {
            steps {
              
                bat "%VENV_DIR%\\Scripts\\pytest --maxfail=1 --disable-warnings -v"
                echo "The DB username: ${env.USERNAME} and the password is: ${env.PASSWORD}"
            }
        }
    }
}
