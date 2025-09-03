pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo "Building the application..."
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying the application..."
            }
        }

        stage('Approval') {  
            steps {
                input message: "Do you want to proceed with deployment?", ok: "Yes, proceed!"
            }
        }
    }
}
