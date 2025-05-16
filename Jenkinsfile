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
                        C:\\Windows\\System32\\OpenSSH\\scp.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" target\\airline-0.0.1-SNAPSHOT.jar %SSH_USER%@13.220.119.113:/home/%SSH_USER%/

                        echo Installing Java...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "if ! command -v java >/dev/null 2>&1; then sudo yum update -y && sudo yum install -y java-17-openjdk; fi"

                        echo Stopping any running Java processes...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "pkill -f 'java -jar' || true"

                        echo Waiting for processes to terminate...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "sleep 5"

                        echo Setting executable permissions...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "chmod +x /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar"

                        echo Starting the application...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "nohup java -jar /home/%SSH_USER%/airline-0.0.1-SNAPSHOT.jar --server.port=8081 --server.address=0.0.0.0 >> /home/%SSH_USER%/airline.log 2>&1 &"

                        echo Waiting for application to start...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "sleep 15"

                        echo Checking application logs...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "cat /home/%SSH_USER%/airline.log"

                        echo Verifying application health...
                        C:\\Windows\\System32\\OpenSSH\\ssh.exe -o ConnectTimeout=30 -o StrictHostKeyChecking=no -i "%SSH_KEY%" %SSH_USER%@13.220.119.113 "for i in {1..5}; do curl -sSf http://localhost:8081/actuator/health && break || sleep 10; done || (echo 'Startup failed' && cat /home/%SSH_USER%/airline.log && exit 1)"

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