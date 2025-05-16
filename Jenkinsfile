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
                        echo Copying JAR to temporary location...
                        C:\\Windows\\System32\\OpenSSH\\scp.exe -i "%SSH_KEY%" target\\airline-0.0.1-SNAPSHOT.jar %SSH_USER%@13.220.119.113:/home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar.new

                        echo Verifying JAR upload...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "ls -lh /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar.new || exit 1"

                        echo Stopping running Java processes...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "pkill -f 'java -jar' || true"

                        echo Waiting for processes to terminate...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "sleep 5"

                        echo Renaming JAR file...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "mv /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar.new /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar"

                        echo Starting the application...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "nohup java -jar /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar --server.port=8081 --server.address=0.0.0.0 >> /home/%SSH_USER%/airline.log 2>&1 &"

                        echo Waiting for application to start...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "sleep 60"

                        echo Verifying application health...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "for i in {1..5}; do curl -sSf http://localhost:8081/actuator/health | grep 'UP' && break || sleep 10; done || (echo 'Startup failed' && tail -n 100 /home/%SSH_USER%/airline.log && exit 1)"
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
                            echo Waiting for application initialization...
                            ping 127.0.0.1 -n 90 > nul

                            echo Testing connectivity...
                            curl -v --retry 5 --retry-delay 10 --max-time 30 http://13.220.119.113:8081/actuator/health | findstr "UP"

                            echo Validating GET /flights?sort=asc...
                            curl -v --retry 5 --retry-delay 10 --max-time 30 "http://13.220.119.113:8081/flights?sort=asc"

                            echo Validating GET /flights/{id}...
                            curl -v --retry 5 --retry-delay 10 --max-time 30 http://13.220.119.113:8081/flights/1

                            echo Creating test data for subsequent operations...
                            curl -v --retry 5 --retry-delay 10 --max-time 30 -X POST -H "Content-Type: application/json" -d "{\\"flightId\\":1,\\"passengerName\\":\\"John Doe\\",\\"seatNumber\\":\\"12A\\"}" http://13.220.119.113:8081/tickets

                            echo Validating GET /tickets/{id}...
                            curl -v --retry 5 --retry-delay 10 --max-time 30 http://13.220.119.113:8081/tickets/1

                            echo Validating DELETE /tickets/{id}...
                            curl -v --retry 5 --retry-delay 10 --max-time 30 -X DELETE http://13.220.119.113:8081/tickets/1
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