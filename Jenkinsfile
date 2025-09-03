pipeline {
    agent any

     environment {
        VENV_DIR = "venv"   
    }

    stages {

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
