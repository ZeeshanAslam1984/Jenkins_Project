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
                    sh 'mkdir -p dist'

                    // Define which files to include
                    def filesToInclude = [
                        'app.py',
                        'requirements.txt',
                        'templates',
                        'static'
                    ]

                    filesToInclude.each { item ->
                        if (sh(script: "test -e ${item}", returnStatus: true) == 0) {
                            sh "cp -r ${item} dist/"
                        }
                    }

                    // Create the zip file
                    sh '''
                        cd dist
                        zip -r ../myapp.zip ./*
                    '''
                    sh 'ls -lh myapp.zip'
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
                        echo "==== Copying ZIP to EC2 ===="
                        scp -i $MY_SSH_KEY -o StrictHostKeyChecking=no myapp.zip $USERNAME@$SERVER_IP:/home/ec2-user/

                        echo "==== Deploying on EC2 ===="
                        ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP << 'EOF'
                            set -e

                            echo "==== Creating app dir ===="
                            mkdir -p ${APP_DIR}
                            mv /home/ec2-user/myapp.zip ${APP_DIR}/
                            cd ${APP_DIR}

                            echo "==== Extracting app ===="
                            unzip -o myapp.zip

                            echo "==== Setting up venv ===="
                            rm -rf ${VENV_DIR}
                            python3 -m venv ${VENV_DIR}
                            source ${VENV_DIR}/bin/activate

                            echo "==== Installing dependencies ===="
                            pip install --upgrade pip setuptools wheel
                            pip install --prefer-binary -r requirements.txt

                            echo "==== Restarting flaskapp.service ===="
                            if sudo systemctl is-active --quiet flaskapp.service; then
                                sudo systemctl restart flaskapp.service
                            else
                                sudo systemctl start flaskapp.service
                            fi

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
        }
        failure {
            echo "❌ Deployment Failed!"
        }
    }
}
