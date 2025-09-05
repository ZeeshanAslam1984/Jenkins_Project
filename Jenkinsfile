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

                            echo "Setting up production virtual environment on EC2 (attempt 1)..."
                            if [ ! -d "${VENV_DIR}" ]; then
                                # Try creating with --upgrade-deps to install pip/setuptools/wheel
                                python3 -m venv ${VENV_DIR} --upgrade-deps || echo "venv --upgrade-deps failed, trying base venv creation..."
                            fi

                            # Check if venv/bin/python exists, if not, force basic creation
                            if [ ! -f "${VENV_DIR}/bin/python" ]; then
                                 echo "venv seems broken, recreating without --upgrade-deps..."
                                 rm -rf ${VENV_DIR}
                                 python3 -m venv ${VENV_DIR}
                            fi

                            echo "Activating production virtual environment..."
                            . ${VENV_DIR}/bin/activate

                            # Explicitly install/upgrade pip, setuptools, wheel INTO the venv
                            # This is the most reliable way to ensure they exist in the correct location
                            echo "Explicitly installing pip, setuptools, wheel into the venv..."
                            python3 -m pip install --target ${VENV_DIR}/lib/python*/site-packages --upgrade pip setuptools wheel

                            # Create symlinks for pip, python if they are missing after the above step
                            PYTHON_VERSION_DIR=\$(ls -1 ${VENV_DIR}/lib/ | grep python)
                            SITE_PACKAGES_DIR="${VENV_DIR}/lib/\${PYTHON_VERSION_DIR}/site-packages"
                            BIN_DIR="${VENV_DIR}/bin"

                            if [ ! -f "\${BIN_DIR}/pip" ] && [ -f "\${SITE_PACKAGES_DIR}/bin/pip" ]; then
                                echo "Creating symlink for pip..."
                                ln -s \${SITE_PACKAGES_DIR}/bin/pip* \${BIN_DIR}/ 2>/dev/null || echo "Symlink for pip failed"
                            fi
                            if [ ! -f "\${BIN_DIR}/python" ]; then
                                 echo "Creating symlink for python..."
                                 ln -s /usr/bin/python3 \${BIN_DIR}/python 2>/dev/null || echo "Symlink for python failed"
                            fi

                            # Final check and activation
                            echo "Re-activating virtual environment to ensure correct PATH..."
                            . ${VENV_DIR}/bin/activate

                            # Verify pip is available
                            which pip || (echo "pip still not found!"; ls -la ${VENV_DIR}/bin; exit 1)

                            echo "Upgrading application dependencies from requirements.txt..."
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
