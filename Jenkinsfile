pipeline {
    agent any

    environment {
        VENV_DIR = "venv"
        SERVER_CREDS = credentials('server-creds') 
    }

    stages {

        stage('Setup') {
    steps {
        echo "my creds: ${SERVER_CREDS}"
        echo "Username: ${SERVER_CREDS_USR}"
        echo "Username: ${SERVER_CREDS_PSW}"
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
