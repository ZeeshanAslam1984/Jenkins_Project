pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
    }

    stages {
        stage('Setup') {
            steps {
                bat 'pip install -r requirements.txt'
            }
        }

        stage('Test') {
            steps {
                bat 'pytest'
            }
        }

        stage('Package code') {
            steps {
                bat 'powershell -Command "Compress-Archive -Path * -DestinationPath myapp.zip"'
                bat 'dir'
            }
        }

        stage('Deploy to Prod') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key',
                                                  keyFileVariable: 'MY_SSH_KEY',
                                                  usernameVariable: 'username')]) {
                    bat '''
scp -i %MY_SSH_KEY% -o StrictHostKeyChecking=no myapp.zip %username%@%SERVER_IP%:/home/ec2-user/
ssh -i %MY_SSH_KEY% -o StrictHostKeyChecking=no %username%@%SERVER_IP% "unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/ && cd /home/ec2-user/app/ && source venv/bin/activate && pip install -r requirements.txt && sudo systemctl restart flaskapp.service"
                    '''
                }
            }
        }
    }
}
