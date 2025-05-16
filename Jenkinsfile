pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Build') {
            steps {
                bat """
                docker run -v "%CD%":/workspace maven:3.9.9-eclipse-temurin-21 /bin/sh -c "cd /workspace && mvn clean package"
                """
            }
        }

        stage('Test') {
            steps {
                bat """
                docker run -v "%CD%":/workspace maven:3.9.9-eclipse-temurin-21 /bin/sh -c "cd /workspace && mvn test"
                """
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                    bat """
                        icacls "%SSH_KEY%" /inheritance:r
                        icacls "%SSH_KEY%" /grant:r "%USERNAME%:F"

                        scp -i "%SSH_KEY%" target\\airline-0.0.1-SNAPSHOT.jar %SSH_USER%@13.220.119.113:/home/%SSH_USER%/

                        ssh -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "
                            sudo amazon-linux-extras enable corretto8
                            sudo yum install -y java-21-amazon-corretto
                            pkill -f 'java -jar' || true
                            sleep 5
                            chmod +x /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar
                            nohup java -jar /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar --server.port=8081 --server.address=0.0.0.0 >> /home/%SSH_USER%/airline.log 2>&1 &
                            sleep 15
                            for i in 1 2 3 4 5; do
                                curl -sSf http://localhost:8081/actuator/health && break
                                sleep 10
                            done || { echo 'Startup failed'; cat /home/%SSH_USER%/airline.log; exit 1; }
                            cat /home/%SSH_USER%/airline.log
                        "
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    retry(5) {
                        timeout(time: 3, unit: 'MINUTES') {
                            bat """
                            ping 127.0.0.1 -n 90 > nul
                            curl -v --retry 5 --retry-delay 10 --max-time 30 http://13.220.119.113:8081/actuator/health
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts 'target/*.jar'
            junit 'target/surefire-reports/*.xml'
        }
        failure {
            withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                bat """
                ssh -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "tail -n 100 /home/%SSH_USER%/airline.log" > deployment.log
                """
            }
            archiveArtifacts artifacts: 'deployment.log', allowEmptyArchive: true
        }
    }
}