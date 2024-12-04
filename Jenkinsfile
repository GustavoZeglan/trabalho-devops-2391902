pipeline {
    agent any

    environment {
        REPOSITORY = 'https://github.com/GustavoZeglan/trabalho-devops-2391902.git'
        BRANCH = 'main'
        CREDENTIAL = 'jenkins-user-github'
    }

    stages {
        stage('Clone the Git repository') {
            steps {
                git branch: "${BRANCH}", credentialsId: "${CREDENTIAL}", url: "${REPOSITORY}"
            }
        }

        stage('Start the containers') {
            steps {
                script {
                    sh '''
                        docker compose --build 
                    '''

                    sh '''
                        docker compose up -d
                    '''
                }
            }
        }

        stage('Make the tests') {
            steps {
                script {
                    sh 'sleep 40' 
                    sh 'docker compose run --rm test'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}
