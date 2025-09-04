pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
        PREINSTALLED_VENV_JENKINS = "/home/vboxuser/jenkins-venv"
        PREINSTALLED_VENV_EC2 = "/home/ec2-user/jenkins-venv"
    }

    stage('Setup') {
    steps {
        sh '''
            #!/bin/bash
            source /home/vboxuser/jenkins-venv/bin/activate
            pip install --upgrade pip
            pip install -r requirements.txt --upgrade --quiet
        '''
    }
}

        stage('Test') {
            steps {
                sh """
                    source ${PREINSTALLED_VENV_JENKINS}/bin/activate
                    pytest
                """
            }
        }

        stage('Package code') {
            steps {
                sh "zip -r myapp.zip ./* -x '*.git*' '${PREINSTALLED_VENV_JENKINS}/*'"
                sh "ls -lart"
            }
        }

        stage('Deploy to Prod') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key', keyFileVariable: 'MY_SSH_KEY', usernameVariable: 'USERNAME')]) {
                    sh """
                        # Copy zip to EC2
                        scp -i \$MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip \${USERNAME}@\${SERVER_IP}:/home/ec2-user/

                        # Deploy on EC2
                        ssh -i \$MY_SSH_KEY -o StrictHostKeyChecking=no \${USERNAME}@\${SERVER_IP} << 'EOF'
                            mkdir -p /home/ec2-user/app
                            unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/
                            cd /home/ec2-user/app
                            # Activate preinstalled venv on EC2
                            source ${PREINSTALLED_VENV_EC2}/bin/activate
                            pip install --upgrade pip
                            pip install -r requirements.txt --upgrade --quiet
                            sudo systemctl restart flaskapp.service
                        EOF
                    """
                }
            }
        }

    } // end of stages
} // end of pipeline
