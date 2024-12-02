pipeline {
    agent any

    environment {
        REPOSITORY = 'https://github.com/GustavoZeglan/trabalho-devops-2391902.git'
        BRANCH = 'main'
    }

    stages {
        stage('Clone the Git repository and build the containers') {
            steps {
                git branch: "${BRANCH}", url: "${REPOSITORY}"
            }
        }

        stage('Start the containers and run the application tests') {
            steps {
                script {
                    sh 'docker-compose up -d mariadb flask test mysqld_exporter prometheus grafana'
                    sh 'sleep 40' 

                    try {
                        sh 'docker-compose run --rm test'
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        error "Application tests failed"
                    }
                }
            }
        }

        stage('Keep the application running') {
            steps {
                script {
                    sh 'docker-compose up -d mariadb flask test mysqld_exporter prometheus grafana'
                }
            }
        }
    }

    post {
        failure {
            sh 'docker-compose down -v'
        }
    }
}
