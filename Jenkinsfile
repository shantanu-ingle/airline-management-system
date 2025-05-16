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

                        echo Starting deployment at %DATE% && time /t

                        echo Copying JAR file with atomic replacement...
                        C:\\Windows\\System32\\OpenSSH\\scp.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" target\\airline-0.0.1-SNAPSHOT.jar %SSH_USER%@13.220.119.113:/home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar.tmp

                        echo Verifying JAR file...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "test -f /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar.tmp && mv /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar.tmp /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar || (echo 'JAR transfer failed' && exit 1)"

                        echo Ensuring Java 21 is installed...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "sudo amazon-linux-extras enable corretto8 && sudo yum install -y java-21-amazon-corretto"

                        echo Stopping previous instance...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "pkill -f 'java -jar airline-0.0.1-SNAPSHOT.jar' || true"

                        echo Waiting for clean shutdown...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "sleep 10"

                        echo Starting application with proper logging...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "nohup java -jar /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar --server.port=8081 --server.address=0.0.0.0 >> /home/%SSH_USER%/airline.log 2>&1 &"

                        echo Waiting for application initialization...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "for i in `seq 1 30`; do curl -sSf http://localhost:8081/actuator/health | grep -q '\"status\":\"UP\"' && break || sleep 2; done"

                        echo Verifying application status...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "curl -sSf http://localhost:8081/actuator/health || (echo 'Application failed to start' && cat /home/%SSH_USER%/airline.log && exit 1)"

                        echo Deployment finished at %DATE% && time /t
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
                ssh -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "tail -n 100 /home/%SSH_USER%/airline.log" > deployment.log
                """
            }
            archiveArtifacts artifacts: 'deployment.log', allowEmptyArchive: true
        }
    }
}