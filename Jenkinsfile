pipeline {
    agent any

    environment {
        SERVER_IP = credentials('prod-server-ip')
    }

    stages {
        stage('Setup') {
            steps {
                bat """
                    python -m venv venv
                    venv\\Scripts\\pip install --upgrade pip
                    venv\\Scripts\\pip install -r requirements.txt
                """
            }
        }

        stage('Test') {
            steps {
                bat "venv\\Scripts\\pytest"
            }
        }

        stage('Package code') {
            steps {
                bat 'powershell -Command "Compress-Archive -Path * -DestinationPath myapp.zip -Force"'
                bat "dir"
            }
        }

 
ssh -i ~/KK-prod.pem ec2-user@3.25.200.84 \
   "unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/ && \
    cd /home/ec2-user/app/ && \
    source venv/bin/activate && \
    pip install -r requirements.txt && \
    sudo systemctl restart flaskapp.service"



    }          

}
