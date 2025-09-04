pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
    }

    stages {

        stage('Setup') {
            steps {
                sh '''#!/bin/bash
                    python3 -m venv .venv
                    source .venv/bin/activate
                    pip install --upgrade pip grpcio Flask -r requirements.txt --quiet
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''#!/bin/bash
                    source .venv/bin/activate
                    pytest
                '''
            }
        }

        stage('Package code') {
            steps {
                sh "zip -r myapp.zip ./* -x '*.git*' '.venv/*'"
                sh "ls -lart"
            }
        }

        stage('Deploy to Prod') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key', keyFileVariable: 'MY_SSH_KEY', usernameVariable: 'USERNAME')]) {
                    sh '''#!/bin/bash
                        scp -i $MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip ${USERNAME}@${SERVER_IP}:/home/ec2-user/
                        ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no ${USERNAME}@${SERVER_IP} << 'EOF'
                            mkdir -p /home/ec2-user/app
                            unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/
                            cd /home/ec2-user/app
                            python3 -m venv .venv
                            source .venv/bin/activate
                            pip install --upgrade pip -r requirements.txt --quiet
                            sudo systemctl restart flaskapp.service
                        EOF
                    '''
                }
            }
        }

    } // end of stages
}
