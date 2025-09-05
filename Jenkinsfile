pipeline {
    agent any

    environment {
        SERVER_IP     = credentials('prod-server-ip')
        SSH_KEY       = credentials('ssh-key')
        USERNAME      = 'ec2-user'
        APP_DIR       = "/home/ec2-user/app"
        VENV_DIR      = "${APP_DIR}/venv"
        SERVICE_NAME  = "flaskapp.service"
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
                    sh 'rm -rf dist myapp.zip'
                    sh 'mkdir -p dist'

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

                    sh '''
                        cd dist
                        zip -r ../myapp.zip ./*
                    '''
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
                        echo "==== Copying files to EC2 with rsync ===="
                        rsync -avz -e "ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no" myapp.zip $USERNAME@$SERVER_IP:/home/ec2-user/

                        echo "==== Running remote deployment script on EC2 ===="
                        ssh -i $MY_SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP 'bash -s' << 'ENDSSH'
                            set -e

                            APP_DIR=/home/ec2-user/app
                            VENV_DIR=$APP_DIR/venv
                            BACKUP_DIR=/home/ec2-user/app_backup
                            SERVICE_NAME=${SERVICE_NAME}

                            echo "==== Creating application directory ===="
                            mkdir -p $APP_DIR

                            echo "==== Backing up old app ===="
                            rm -rf $BACKUP_DIR
                            if [ -d "$APP_DIR" ]; then
                                cp -r $APP_DIR $BACKUP_DIR
                            fi

                            echo "==== Moving ZIP file ===="
                            mv /home/ec2-user/myapp.zip $APP_DIR/
                            cd $APP_DIR

                            echo "==== Extracting application code ===="
                            unzip -o myapp.zip

                            echo "==== Setting up production virtual environment ===="
                            rm -rf $VENV_DIR
                            python3 -m venv $VENV_DIR
                            source $VENV_DIR/bin/activate

                            echo "==== Ensuring pip is available ===="
                            python3 -m pip install --upgrade pip setuptools wheel

                            echo "==== Installing grpcio explicitly ===="
                            pip install --prefer-binary grpcio==1.74.0 grpcio-status==1.74.0

                            echo "==== Installing application dependencies ===="
                            pip install --prefer-binary -r requirements.txt

                            echo "==== Restarting service: $SERVICE_NAME ===="
                            if sudo systemctl is-active --quiet $SERVICE_NAME; then
                                sudo systemctl restart $SERVICE_NAME
                            else
                                sudo systemctl start $SERVICE_NAME
                            fi

                            if ! sudo systemctl is-active --quiet $SERVICE_NAME; then
                                echo "❌ ERROR: $SERVICE_NAME failed to start! Rolling back..."
                                sudo systemctl stop $SERVICE_NAME || true
                                rm -rf $APP_DIR
                                mv $BACKUP_DIR $APP_DIR
                                sudo systemctl start $SERVICE_NAME || true
                                exit 1
                            fi

                            echo "✅ Deployment completed successfully!"
                            rm -rf $BACKUP_DIR
                        ENDSSH
                    '''
                }
            }
        }
    }
}
