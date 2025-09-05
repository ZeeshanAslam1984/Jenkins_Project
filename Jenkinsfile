pipeline {
    agent any

    environment {
        SERVER_IP   = credentials('prod-server-ip')
        SSH_KEY     = credentials('ssh-key')
        APP_DIR     = "/home/ec2-user/app"
        VENV_DIR    = "${APP_DIR}/venv"
        APP_SCRIPT  = "app.py"  // Main app entry point
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
                    // Define what to include in deployment
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
                        'tasks.txt'
                    ]

                    // Create clean dist directory
                    sh 'rm -rf dist && mkdir -p dist'
                    
                    // Copy only required files
                    filesToInclude.each { file ->
                        if (sh(script: "test -f ${file}", returnStatus: true) == 0) {
                            sh "cp ${file} dist/"
                        }
                    }
                    
                    // Copy templates/ if exists
                    if (sh(script: "test -d templates", returnStatus: true) == 0) {
                        sh "cp -r templates dist/"
                    }

                    // Create zip from dist
                    sh '''
                        cd dist && zip -r ../myapp.zip ./
                    '''
                    
                    sh 'ls -lart myapp.zip'
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
                    // Copy zip to EC2
                    sh '''
                        scp -i $MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip $USERNAME@$SERVER_IP:/home/ec2-user/
                    '''

                    // Run remote setup and restart
                    sh '''
                        ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP << 'EOF'
                            set -e  # Exit on any error

                            echo "Creating app directory..."
                            mkdir -p ${APP_DIR}

                            echo "Extracting new version..."
                            unzip -o /home/ec2-user/myapp.zip -d ${APP_DIR}/

                            cd ${APP_DIR}

                            echo "Setting up virtual environment..."
                            if [ ! -d "${VENV_DIR}" ]; then
                                python3 -m venv ${VENV_DIR}
                            fi

                            echo "Activating virtual environment..."
                            source ${VENV_DIR}/bin/activate

                            echo "Upgrading pip..."
                            pip install --upgrade pip setuptools wheel

                            echo "Installing dependencies..."
                            pip install --prefer-binary -r requirements.txt

                            echo "Checking systemd service..."
                            if systemctl is-active --quiet flaskapp.service; then
                                echo "Restarting flaskapp.service..."
                                sudo systemctl restart flaskapp.service
                            else
                                echo "Starting flaskapp.service..."
                                sudo systemctl start flaskapp.service
                            fi

                            # Final status check
                            if ! systemctl is-active --quiet flaskapp.service; then
                                echo "❌ flaskapp.service failed to start!"
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
            echo "🚀 Deployment completed successfully!"
            emailext(
                subject: "Deployment Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "The deployment to production was successful.\n\nView build: ${env.BUILD_URL}",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        failure {
            echo "❌ Deployment failed!"
            emailext(
                subject: "Deployment Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Check console output: ${env.BUILD_URL}",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }
}
