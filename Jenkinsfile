pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')  // EC2 public IP
    }

    stages {
        stage('Setup') {
            steps {
                sh '''
                python3 -m venv j-venv
                . j-venv/bin/activate
                pip install --upgrade pip setuptools wheel
                pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                . j-venv/bin/activate
                pytest
                '''
            }
        }

        stage('Package code') {
            steps {
                sh "zip -r myapp.zip ./* -x '*.git*'"
                sh "ls -lart"
            }
        }

        stage('Deploy to Prod') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key', keyFileVariable: 'MY_SSH_KEY', usernameVariable: 'USERNAME')]) {
                    sh """
                    scp -i \$MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip \$USERNAME@\$SERVER_IP:/home/ec2-user/

                    ssh -i \$MY_SSH_KEY -o StrictHostKeyChecking=no \$USERNAME@\$SERVER_IP << 'EOF'
                        mkdir -p /home/ec2-user/app
                        unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/
                        cd /home/ec2-user/app/
                        if [ ! -d "venv" ]; then
                            python3 -m venv venv
                        fi
                        . venv/bin/activate
                        pip install --upgrade pip
                        pip install -r requirements.txt
                        sudo systemctl restart flaskapp.service
                    EOF
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}
