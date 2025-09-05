pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
        SSH_KEY   = credentials('ssh-key')
        USERNAME  = 'ec2-user'
        APP_DIR   = "/home/ec2-user/app"
        VENV_DIR  = "${APP_DIR}/venv"
    }

    stages {
        stage('Setup & Install') {
            steps {
                sh '''
                    python3 -m venv j-venv
                    . j-venv/bin/activate
                    pip install --upgrade pip setuptools wheel
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    . j-venv/bin/activate
                    pytest --verbose
                '''
            }
        }

        stage('Package Code') {
            steps {
                script {
                    sh 'rm -rf dist myapp.zip'
                    sh 'mkdir -p dist'

                    def filesToInclude = [
                        'app.py',
                        'requirements.txt',
                        'Jenkinsfile',
                        'README.md',
                        'Dockerfile',
                        'script.sh',
                        'deployment.yaml',
                        'deployment_production.yaml',
                        'eksClusterCreation.md',
                        'commands.txt',
                        'tasks.txt',
                        'templates'
                    ]

                    filesToInclude.each { item ->
                        if (sh(script: "test -e ${item}", returnStatus: true) == 0) {
                            sh "cp -r ${item} dist/"
                        }
                    }

                    sh 'cd dist && zip -r ../myapp.zip .'
                    sh 'ls -l myapp.zip'
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ssh-key',
                    keyFileVariable: 'MY_SSH_KEY',
                    usernameVariable: 'USERNAME'
                )]) {
                    sh '''
                        scp -i $MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip $USERNAME@$SERVER_IP:/home/ec2-user/
                    '''

                    sh '''
                        ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP << EOF
                            set -e
                            echo "Creating app directory..."
                            mkdir -p ${APP_DIR}
                            mv /home/ec2-user/myapp.zip ${APP_DIR}/
                            cd ${APP_DIR}

                            echo "Extracting code..."
                            unzip -o myapp.zip

                            echo "Setting up virtual environment..."
                            if [ ! -d "${VENV_DIR}" ]; then
                                python3 -m venv ${VENV_DIR}
                            fi

                            echo "Activating virtual environment..."
                            . ${VENV_DIR}/bin/activate

                            echo "Upgrading pip..."
                            pip install --upgrade pip setuptools wheel

                            echo "Installing dependencies..."
                            pip install --prefer-binary -r requirements.txt

                            echo "Restarting flaskapp.service..."
                            if sudo systemctl is-active --quiet flaskapp.service; then
                                sudo systemctl restart flaskapp.service
                            else
                                sudo systemctl start flaskapp.service
                            fi

                            if ! sudo systemctl is-active --quiet flaskapp.service; then
                                echo "❌ Service failed to start!"
                                sudo systemctl status flaskapp.service --no-pager
                                exit 1
                            fi

                            echo "✅ Deployment successful!"
                        EOF
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🚀 Deployment succeeded!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
