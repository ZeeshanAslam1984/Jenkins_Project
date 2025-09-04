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

        stage('Deploy to Prod') {
    steps {
        bat """
            echo Copying zip file to EC2...
            pscp -i C:\\path\\to\\private-key.ppk -batch myapp.zip "ec2-user@%SERVER_IP%:/home/ec2-user/"

            echo Deploying on EC2...
            plink -i C:\Users\Zeeshan\Downloads\KK-prod.ppk ec2-user@3.25.200.84 ^
                "unzip -o /home/ec2-user/myapp.zip -d /home/ec2-user/app/ && ^
                cd /home/ec2-user/app/ && ^
                source venv/bin/activate && ^
                pip install -r requirements.txt && ^
                sudo systemctl restart flaskapp.service"
        """
    }
}

    }          

}
