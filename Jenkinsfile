pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
    }

    stages {
        stage('Setup') {
            steps {
                bat """
                    python -m venv venv
                    venv\\Scripts\\pip install --upgrade pip
                    venv\\Scripts\\pip install -r requirements.txt
                """
            }
        }

        stage('Test') {
            steps {
                bat "venv\\Scripts\\pytest"
            }
        }

        stage('Package code') {
            steps {
                bat 'powershell -Command "Compress-Archive -Path * -DestinationPath myapp.zip -Force"'
                bat "dir"
            }
        }

        stage('Deploy to Prod') {
            steps {
                // Load SSH credentials stored in Jenkins
                sshagent (credentials: ['MY_SSH_KEY_ID']) {
                    bat """
                        REM 🚀 Copy zip file to EC2
                        scp -o StrictHostKeyChecking=no myapp.zip ec2-user@%SERVER_IP%:/home/ec2-user/

                        REM 🚀 SSH into EC2 and deploy
                        ssh -o StrictHostKeyChecking=no ec2-user@%SERVER_IP% "unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/ && cd /home/ec2-user/app/ && source venv/bin/activate && pip install -r requirements.txt && sudo systemctl restart flaskapp.service"
                    """
                }
            }
        }
    }
}
