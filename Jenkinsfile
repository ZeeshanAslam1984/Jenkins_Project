pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')  // EC2 public IP
    }

    stages {
        stage('Setup') {
            steps {
                // Create a virtual environment if it doesn't exist
                sh '''
                python3 -m venv j-venv
                source j-venv/bin/activate
                pip install --upgrade pip setuptools wheel
                pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                source j-venv/bin/activate
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
                // Use SSH key from Jenkins credentials
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key', keyFileVariable: 'MY_SSH_KEY', usernameVariable: 'USERNAME')]) {
                    sh """
                    # Copy zip to EC2
                    scp -i \$MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip \$USERNAME@\$SERVER_IP:/home/ec2-user/

                    # SSH into EC2 and deploy
                    ssh -i \$MY_SSH_KEY -o StrictHostKeyChecking=no \$USERNAME@\$SERVER_IP << 'EOF'
                        # Create app folder if not exists
                        mkdir -p /home/ec2-user/app
                        
                        # Unzip project
                        unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/

                        cd /home/ec2-user/app/

                        # Create virtual environment if not exists
                        if [ ! -d "venv" ]; then
                            python3 -m venv venv
                        fi

                        # Activate venv and install dependencies
                        source venv/bin/activate
                        pip install --upgrade pip
                        pip install -r requirements.txt

                        # Restart Flask service
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
