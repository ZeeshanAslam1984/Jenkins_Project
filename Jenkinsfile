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
                    // Clean up any previous artifacts
                    sh 'rm -rf dist myapp.zip'

                    // Create a clean directory for deployment files
                    sh 'mkdir -p dist'

                    // Define the list of files/directories to include
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

                    // Copy each specified file/directory if it exists
                    filesToInclude.each { item ->
                        if (sh(script: "test -e ${item}", returnStatus: true) == 0) {
                            sh "cp -r ${item} dist/"
                        }
                    }

                    // Create the ZIP archive from the dist directory
                    sh 'cd dist && zip -r ../myapp.zip .'

                    // Display the size of the created ZIP file
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
                    // Copy the ZIP file to the EC2 instance
                    sh '''
                        scp -i $MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip $USERNAME@$SERVER_IP:/home/ec2-user/
                    '''

                    // Execute deployment commands on the EC2 instance
                    sh '''
                        ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP << EOF
                            set -e # Exit immediately if a command exits with a non-zero status

                            echo "Creating application directory..."
                            mkdir -p ${APP_DIR}

                            echo "Moving ZIP file to application directory..."
                            mv /home/ec2-user/myapp.zip ${APP_DIR}/
                            cd ${APP_DIR}

                            echo "Extracting application code..."
                            unzip -o myapp.zip

                            echo "Setting up virtual environment..."
                            if [ ! -d "${VENV_DIR}" ]; then
                                python3 -m venv ${VENV_DIR}
                            fi

                            echo "Activating virtual environment..."
                            . ${VENV_DIR}/bin/activate

                            echo "Ensuring pip is installed in the virtual environment..."
                            # Use python3 explicitly for Amazon Linux 2023
                            python3 -m ensurepip --upgrade || echo "Warning: ensurepip failed, attempting alternative method..."

                            # Fallback: Download and run get-pip.py if pip is still not available
                            if ! command -v pip &> /dev/null; then
                                echo "pip not found, downloading get-pip.py..."
                                curl -s https://bootstrap.pypa.io/get-pip.py -o /tmp/get-pip.py
                                python3 /tmp/get-pip.py --user
                                # Ensure the user's local bin is in PATH for subsequent commands in this session
                                export PATH=\$HOME/.local/bin:\$PATH
                            fi

                            echo "Upgrading pip, setuptools, and wheel..."
                            pip install --upgrade pip setuptools wheel

                            echo "Installing application dependencies from requirements.txt..."
                            pip install --prefer-binary -r requirements.txt

                            echo "Restarting flaskapp.service..."
                            if sudo systemctl is-active --quiet flaskapp.service; then
                                echo "Service is active, restarting..."
                                sudo systemctl restart flaskapp.service
                            else
                                echo "Service not active, starting..."
                                sudo systemctl start flaskapp.service
                            fi

                            # Final verification that the service is running
                            if ! sudo systemctl is-active --quiet flaskapp.service; then
                                echo "❌ ERROR: flaskapp.service failed to start!"
                                sudo systemctl status flaskapp.service --no-pager
                                exit 1
                            fi

                            echo "✅ Deployment completed successfully!"
                        EOF
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🚀 Deployment Succeeded!"
            // Optionally add email notifications here
            // emailext(subject: "SUCCESS: Pipeline \${env.JOB_NAME} [\${env.BUILD_NUMBER}]", body: "See \${env.BUILD_URL}", recipientProviders: [[$class: 'DevelopersRecipientProvider']])
        }
        failure {
            echo "❌ Deployment Failed!"
            // Optionally add email notifications here
            // emailext(subject: "FAILURE: Pipeline \${env.JOB_NAME} [\${env.BUILD_NUMBER}]", body: "See \${env.BUILD_URL}", recipientProviders: [[$class: 'DevelopersRecipientProvider']])
        }
    }
}
