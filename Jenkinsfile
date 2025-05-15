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

                        echo Copying JAR file...
                        C:\\Windows\\System32\\OpenSSH\\scp.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" target\\airline-0.0.1-SNAPSHOT.jar %SSH_USER%@54.159.204.82:/home/%SSH_USER%/

                        echo Deploying application...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@54.159.204.82 "pkill -f 'java -jar' || true && sleep 5 && export SPRING_PROFILES_ACTIVE=production && nohup java -jar /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar --server.port=8081 --server.address=0.0.0.0 > /home/%SSH_USER%/app.log 2>&1 & sleep 30 && cat /home/%SSH_USER%/app.log && curl -sSf http://localhost:8081/actuator/health || (echo 'Startup failed' && exit 1)"

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
                            echo "Waiting for application to be fully ready..."
                            ping 127.0.0.1 -n 60 > nul

                            echo "Testing base endpoint..."
                            curl -v --retry 5 --retry-delay 10 --max-time 30 http://54.159.204.82:8081/actuator/health

                            echo "Validating GET /flights?sort=asc..."
                            curl -v --retry 5 --retry-delay 10 --max-time 30 "http://54.159.204.82:8081/flights?sort=asc"

                            echo "Validating GET /flights/{id}..."
                            curl -v --retry 5 --retry-delay 10 --max-time 30 http://54.159.204.82:8081/flights/1

                            echo "Creating test data for subsequent operations..."
                            curl -v --retry 5 --retry-delay 10 --max-time 30 -X POST -H "Content-Type: application/json" -d "{\\"flightId\\":1,\\"passengerName\\":\\"John Doe\\",\\"seatNumber\\":\\"12A\\"}" http://54.159.204.82:8081/tickets

                            echo "Validating GET /tickets/{id}..."
                            curl -v --retry 5 --retry-delay 10 --max-time 30 http://54.159.204.82:8081/tickets/1

                            echo "Validating DELETE /tickets/{id}..."
                            curl -v --retry 5 --retry-delay 10 --max-time 30 -X DELETE http://54.159.204.82:8081/tickets/1
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
                ssh -i "%SSH_KEY%" %SSH_USER%@54.159.204.82 "cat /home/%SSH_USER%/app.log" > deployment.log
                """
            }
            archiveArtifacts artifacts: 'deployment.log', allowEmptyArchive: true
        }
    }
}