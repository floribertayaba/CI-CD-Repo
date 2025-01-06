pipeline {  
    agent { label 'DevOps-slave1' }

    environment {
        ECR_REPO = 'your-ecr-repo-url'
        IMAGE_NAME = 'app-image'
        TAG = "${env.BRANCH_NAME}-${env.BUILD_ID}"
        SSH_KEY = credentials('ec2-ssh-key')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: 'https://github.com/your-org/your-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${env.ECR_REPO}:${env.TAG}")
                }
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-ecr', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh """
                        aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${env.ECR_REPO}
                        docker push ${env.ECR_REPO}:${env.TAG}
                    """
                }
            }
            post {
                success {
                    emailext(
                        subject: "Jenkins Job - Docker Image Pushed to ECR Successfully",
                        body: """
                            Hello,

                            The Docker image '${env.IMAGE_NAME}:${env.TAG}' has been successfully pushed to ECR.

                            Best regards,
                            Jenkins
                        """,
                        recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                        to: "m.ehtasham.azhar@gmail.com"
                    )
                }
            }
        }

        stage('Static Code Analysis - SonarQube') {
            steps {
                script {
                    withSonarQubeEnv('SonarQubeServer') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage('Container Security Scan - Trivy') {
            steps {
                script {
                    sh "trivy image ${env.ECR_REPO}:${env.TAG}"
                }
            }
        }

        stage('Deploy to Environment') {
            steps {
                script {
                    def targetHost = ''
                    if (env.BRANCH_NAME == 'dev') {
                        targetHost = '<DEV-EC2-IP>'
                    } else if (env.BRANCH_NAME == 'staging') {
                        targetHost = '<STAGING-EC2-IP>'
                    } else if (env.BRANCH_NAME == 'main') {
                        targetHost = '<PROD-EC2-IP>'
                    }

                    sh """
                    ssh -i ${SSH_KEY} ec2-user@${targetHost} << EOF
                    docker pull ${env.ECR_REPO}:${env.TAG}
                    docker stop ${env.IMAGE_NAME} || true
                    docker rm ${env.IMAGE_NAME} || true
                    docker run -d --name ${env.IMAGE_NAME} -p 80:80 ${env.ECR_REPO}:${env.TAG}
                    EOF
                    """
                }
            }
        }
    }

    post {
        always {
            node('DevOps-slave1') { // Specify the label for the node
                cleanWs() // Clean up workspace
            }
        }
    }
}

