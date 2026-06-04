pipeline {
    agent { label 'anett-ec2' }

    environment {
        DOCKERHUB_USER = 'annevarga'
        SONAR_URL      = 'https://sonarqube.devopspro.cloud'
    }

    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'master', url: 'https://github.com/annevarga/devops-swiggy-project.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        /home/ubuntu/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=swiggy-anett \
                          -Dsonar.projectName=swiggy-anett \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_URL} \
                          -Dsonar.token=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Install Dependencies') {
            steps { sh "npm install" }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh "trivy fs . --exit-code 0 --severity HIGH,CRITICAL -f table -o trivy-fs-report.txt"
                archiveArtifacts artifacts: 'trivy-fs-report.txt', allowEmptyArchive: true
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-creds-anett', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker build -t swiggy .
                        docker tag swiggy annevarga/swiggy:latest
                        docker push annevarga/swiggy:latest
                    """
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image annevarga/swiggy:latest --exit-code 0 --severity HIGH,CRITICAL -f table -o trivy-image-report.txt"
                archiveArtifacts artifacts: 'trivy-image-report.txt', allowEmptyArchive: true
            }
        }

        stage('Deploy to Container') {
            steps {
                sh """
                    docker rm -f swiggy || true
                    docker run -d --name swiggy -p 3000:3000 annevarga/swiggy:latest
                """
            }
        }
    }

    post {
        always { echo "Pipeline execution completed!" }
        failure { echo "Pipeline failed. Check logs for details." }
        success { echo "Swiggy App deployed at http://32.192.4.93:3000" }
    }
}
