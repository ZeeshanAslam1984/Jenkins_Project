pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
        SSH_KEY   = credentials('ssh-key')  // Private key from Jenkins credentials
        APP_DIR   = "/home/ec2-user/app"
        VENV_DIR  = "${APP_DIR}/venv"
    }

    stages {
        stage('Setup Local') {
            steps {
                sh '''
                    python3 -m venv j-venv
                    . j-venv/bin/activate
                    pip install --upgrade pip setuptools wheel
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Test Local') {
            steps {
                sh '''
                    . j-venv/bin/activate
                    pytest
                '''
            }
        }

        stage('Package Code') {
            steps {
                sh "zip -r myapp.zip ./* -x '*.git*'"
                sh "ls -lart"
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key', keyFileVariable: 'MY_SSH_KEY', usernameVariable: 'USERNAME')]) {
                    sh """
                        scp -i \$MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip \$USERNAME@\$SERVER_IP:/home/ec2-user/

                        ssh -i \$MY_SSH_KEY -o StrictHostKeyChecking=no \$USERNAME@\$SERVER_IP << EOF
                            mkdir -p ${APP_DIR}
                            unzip -o /home/ec2-user/myapp.zip -d ${APP_DIR}/
                            cd ${APP_DIR}
                            if [ ! -d "${VENV_DIR}" ]; then
                                python3 -m venv ${VENV_DIR}
                            fi
                            . ${VENV_DIR}/bin/activate
                            pip install --upgrade pip setuptools wheel
                            pip install --prefer-binary -r requirements.txt
                            sudo systemctl restart flaskapp.service || echo "Service not found, skipping restart"
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
