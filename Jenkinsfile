pipeline {
    agent any

    stages {
        stage('Clone the Git repository and build the containers') {
            steps {
                script {
                    git branch: "main", url: "https://github.com/vinic1u/Trabalho_DevOps_2392132.git"
                    sh 'docker-compose down -v'
                    sh 'docker-compose build'
                }
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