pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
        SSH_KEY   = credentials('ssh-key')
        USERNAME  = 'ec2-user'
        APP_DIR   = "/home/ec2-user/app"
        VENV_DIR  = "${APP_DIR}/venv" // Production venv on EC2
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
                    // --- CRITICAL FIX: Exclude local j-venv from deployment ---
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

                    // --- CRITICAL FIX: Explicitly exclude j-venv when creating ZIP ---
                    // Create the ZIP archive from the dist directory contents
                    sh '''
                        cd dist
                        zip -r ../myapp.zip ./*
                    '''

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

                            echo "Creating application directory on EC2..."
                            mkdir -p ${APP_DIR}

                            echo "Moving ZIP file to application directory..."
                            mv /home/ec2-user/myapp.zip ${APP_DIR}/
                            cd ${APP_DIR}

                            echo "Extracting application code..."
                            unzip -o myapp.zip

                            echo "Setting up production virtual environment on EC2..."
                            if [ ! -d "${VENV_DIR}" ]; then
                                python3 -m venv ${VENV_DIR}
                            fi

                            echo "Activating production virtual environment..."
                            . ${VENV_DIR}/bin/activate

                            echo "Ensuring pip is installed in the EC2 virtual environment..."
                            # Use python3 explicitly for Amazon Linux 2023
                            python3 -m ensurepip --upgrade || echo "Warning: ensurepip failed, attempting alternative method..."

                            # Fallback: Download and run get-pip.py if pip is still not available or not in venv
                            if [ ! -f "${VENV_DIR}/bin/pip" ]; then
                                echo "pip binary not found in EC2 venv, downloading get-pip.py..."
                                curl -s https://bootstrap.pypa.io/get-pip.py -o /tmp/get-pip.py
                                python3 /tmp/get-pip.py --force-reinstall --target=${VENV_DIR}/lib/python*/site-packages
                                # Create a symlink for pip in the venv bin directory if it wasn't created
                                if [ ! -f "${VENV_DIR}/bin/pip" ]; then
                                   echo "Creating symlink for pip (if needed)..."
                                   ln -s ${VENV_DIR}/lib/python*/site-packages/bin/pip* ${VENV_DIR}/bin/ 2>/dev/null || echo "Symlink creation failed or not needed"
                                fi
                            fi

                            # Explicitly reactivate to ensure PATH and environment are correct for the venv's pip
                            echo "Re-activating virtual environment to ensure correct PATH..."
                            . ${VENV_DIR}/bin/activate

                            # Verify pip is available in the PATH
                            which pip || echo "pip not found in PATH after activation"

                            echo "Upgrading pip, setuptools, and wheel using full path..."
                            # Use the full path to pip to be absolutely sure
                            ${VENV_DIR}/bin/pip install --upgrade pip setuptools wheel

                            echo "Installing application dependencies from requirements.txt..."
                            ${VENV_DIR}/bin/pip install --prefer-binary -r requirements.txt

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
