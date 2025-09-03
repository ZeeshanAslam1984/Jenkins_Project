pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                
            }
        }

        stage('Deploy') {
            steps {
               
            }
        }

        stage('Approval') {  
            steps {
                input message: "Do you want to proceed with deployment?", ok: "Yes, proceed!"
            }
        }
    }
}
