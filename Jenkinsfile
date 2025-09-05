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
                    source j-venv/bin/activate
                    pip install --upgrade pip setuptools wheel
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    source j-venv/bin/activate
                    pytest --verbose
                '''
            }
        }

        stage('Package Code') {
            steps {
                script {
                    // Clean up previous builds
                    sh 'rm -rf dist myapp.zip'

                    // Create a clean distribution directory
                    sh 'mkdir -p dist'

                    // List of files and directories to include
                    def include = [
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

                    // Copy each file/directory if it exists
                    include.each { item ->
                        if (sh(script: "test -e ${item}", returnStatus: true) == 0) {
                            sh "cp -r ${item} dist/"
                        }
                    }

                    // Create ZIP archive
                    sh 'cd dist && zip -r ../myapp.zip .'

                    // Show zip size
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
                    // Copy ZIP to EC2
                    sh '''
                        scp -i $MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip $USERNAME@$SERVER_IP:/home/ec2-user/
                    '''

                    // Run deployment commands on EC2
                    // Note: Use EOF (not 'EOF') so Jenkins expands $APP_DIR, $VENV_DIR
                    sh '''
                        ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP << EOF
                            set -e  # Exit on any error

                            echo "Creating app directory..."
                            mkdir -p ${APP_DIR}

                            echo "Removing old zip if exists..."
                            rm -f /home/ec2-user/myapp.zip

                            echo "Moving new zip to app dir..."
                            mv /home/ec2-user/myapp.zip ${APP_DIR}/
                            cd ${APP_DIR}

                            echo "Extracting code..."
                            unzip -o myapp.zip

                            echo "Setting up virtual environment..."
                            if [ ! -d "${VENV_DIR}" ]; then
                                python3 -m venv ${VENV_DIR}
                            fi

                            echo "Activating virtual environment..."
                            source ${VENV_DIR}/bin/activate

                            echo "Upgrading pip..."
                            pip install --upgrade pip setuptools wheel

                            echo "Installing application dependencies..."
                            pip install --prefer-binary -r requirements.txt

                            echo "Restarting flaskapp.service..."
                            if sudo systemctl is-active --quiet flaskapp.service; then
                                sudo systemctl restart flaskapp.service
                            else
                                sudo systemctl start flaskapp.service
                            fi

                            # Final status check
                            if ! sudo systemctl is-active --quiet flaskapp.service; then
                                echo "❌ Service failed to start!"
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
            echo "🚀 Deployment succeeded!"
            emailext(
                subject: "✅ Deployment Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "The app was successfully deployed to production.\n\nView build: ${env.BUILD_URL}",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        failure {
            echo "❌ Deployment failed!"
            emailext(
                subject: "❌ Deployment Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Check console output: ${env.BUILD_URL}",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }
}
