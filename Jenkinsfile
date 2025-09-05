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
        stage('Setup & Install Locally') {
            steps {
                sh '''
                    python3 -m venv j-venv
                    . j-venv/bin/activate
                    pip install --upgrade pip setuptools wheel
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Local Tests') {
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
                        // Add other necessary files/dirs here
                        'templates' // Example, add others as needed
                    ]

                    // Copy each specified file/directory if it exists
                    filesToInclude.each { item ->
                        if (sh(script: "test -e ${item}", returnStatus: true) == 0) {
                            sh "cp -r ${item} dist/"
                        }
                    }

                    // Create the ZIP archive from the dist directory contents
                    // This explicitly lists files to avoid including j-venv
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
                    // Using 'bash -c' to ensure a proper shell environment
                    sh '''
                        ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP 'bash -c "
                            set -e # Exit immediately if a command exits with a non-zero status

                            echo \"==== Creating application directory on EC2 ==== \"
                            mkdir -p ${APP_DIR}

                            echo \"==== Moving ZIP file ==== \"
                            mv /home/ec2-user/myapp.zip ${APP_DIR}/
                            cd ${APP_DIR}

                            echo \"==== Extracting application code ==== \"
                            unzip -o myapp.zip

                            echo \"==== Setting up production virtual environment ==== \"
                            if [ ! -d \"${VENV_DIR}\" ]; then
                                echo \"Creating new venv...\"
                                python3 -m venv ${VENV_DIR}
                            else
                                echo \"Venv exists, recreating to ensure cleanliness...\"
                                rm -rf ${VENV_DIR}
                                python3 -m venv ${VENV_DIR}
                            fi

                            echo \"==== Activating production virtual environment ==== \"
                            source ${VENV_DIR}/bin/activate

                            echo \"==== Ensuring pip is available in venv ==== \"
                            # Use the system pip to install pip INTO the venv
                            # This is often the most reliable way on AL2023
                            python3 -m pip install --upgrade --target ${VENV_DIR}/lib/python*/site-packages pip

                            # Create a symlink for pip in the venv bin if it doesn't exist
                            if [ ! -f \"${VENV_DIR}/bin/pip\" ]; then
                                PIP_EXECUTABLE=\$(find ${VENV_DIR}/lib -name \"pip\" -type f -executable 2>/dev/null | head -n 1)
                                if [ -n \"\${PIP_EXECUTABLE}\" ]; then
                                    echo \"Creating symlink for pip...\"
                                    ln -s \${PIP_EXECUTABLE} ${VENV_DIR}/bin/pip
                                else
                                    echo \"ERROR: Could not find pip executable to symlink after installation.\"
                                    exit 1
                                fi
                            fi

                            echo \"==== Verifying pip location ==== \"
                            source ${VENV_DIR}/bin/activate # Reactivate to be sure
                            which pip
                            pip --version

                            echo \"==== Installing application dependencies ==== \"
                            pip install --upgrade pip setuptools wheel
                            pip install --prefer-binary -r requirements.txt

                            echo \"==== Restarting flaskapp.service ==== \"
                            if sudo systemctl is-active --quiet flaskapp.service; then
                                echo \"Service is active, restarting...\"
                                sudo systemctl restart flaskapp.service
                            else
                                echo \"Service not active, starting...\"
                                sudo systemctl start flaskapp.service
                            fi

                            # Final verification that the service is running
                            if ! sudo systemctl is-active --quiet flaskapp.service; then
                                echo \"❌ ERROR: flaskapp.service failed to start!\"
                                sudo systemctl status flaskapp.service --no-pager
                                exit 1
                            fi

                            echo \"✅ Deployment completed successfully!\"
                        "'
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🚀 Deployment Succeeded!"
        }
        failure {
            echo "❌ Deployment Failed!"
        }
    }
}
