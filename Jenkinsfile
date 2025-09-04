pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
        VENV_DIR  = "venv"
    }

    stages {
        stage('Setup venv') {
            steps {
                bat """
                    python -m venv %VENV_DIR%
                    call %VENV_DIR%\\Scripts\\activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                """
            }
        }

        stage('Test') {
            steps {
                bat """
                    call %VENV_DIR%\\Scripts\\activate
                    pytest
                """
            }
        }

       stage('Package code') {
    steps {
        bat 'powershell -Command "Compress-Archive -Path * -DestinationPath myapp.zip -Force"'
        bat 'dir'
    }
}

        stage('Deploy to Prod') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key',
                                                  keyFileVariable: 'MY_SSH_KEY',
                                                  usernameVariable: 'username')]) {
                    bat """
:: Fix SSH key permissions (like chmod 600 on Linux)
icacls %MY_SSH_KEY% /inheritance:r /grant:r "%USERNAME%:R"

:: Copy the package to EC2
scp -i %MY_SSH_KEY% -o StrictHostKeyChecking=no myapp.zip %username%@%SERVER_IP%:/home/ec2-user/

:: Deploy on EC2
ssh -i %MY_SSH_KEY% -o StrictHostKeyChecking=no %username%@%SERVER_IP% "unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/ && cd /home/ec2-user/app/ && source venv/bin/activate && pip install -r requirements.txt && sudo systemctl restart flaskapp.service"
                    """
                }
            }
        }
    }
}
