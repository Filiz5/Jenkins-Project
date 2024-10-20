pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = '975049995227'
        AWS_REGION = 'us-east-1'
        ECR_REPO_NAME1 = 'nodejs-image'
        ECR_REPO_NAME2 = 'postgres-image'
        ECR_REPO_NAME3 = 'react-images'
        DOCKER_IMAGE_TAG = "${env.BUILD_NUMBER}"
        EC2_INSTANCE_ID = '' // EC2 instance ID'si daha sonra atanacak
    }

    stages {
        stage('Create Infra') {
            steps {
                script {
                    // Terraform key ve altyapı oluşturma
                    sh '''
                    terraform init
                    terraform apply -auto-approve
                    '''
                }
            }
        }

        stage('Create ECR Repos') {
            steps {
                script {
                    // Her ECR reposu için repo oluşturma
                    sh '''
                    aws ecr create-repository --repository-name ${ECR_REPO_NAME1} --region ${AWS_REGION} || echo "Repo 1 zaten mevcut"
                    aws ecr create-repository --repository-name ${ECR_REPO_NAME2} --region ${AWS_REGION} || echo "Repo 2 zaten mevcut"
                    aws ecr create-repository --repository-name ${ECR_REPO_NAME3} --region ${AWS_REGION} || echo "Repo 3 zaten mevcut"
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    // AWS ECR login işlemi
                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    // App1 için Docker image oluşturma ve ECR'a gönderme
                    dir('nodejs') {
                        sh '''
                        docker build -t ${ECR_REPO_NAME1}:${DOCKER_IMAGE_TAG} .
                        docker tag ${ECR_REPO_NAME1}:${DOCKER_IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME1}:${DOCKER_IMAGE_TAG}
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME1}:${DOCKER_IMAGE_TAG}
                        '''
                    }

                    // App2 için Docker image oluşturma ve ECR'a gönderme
                    dir('postgresql') {
                        sh '''
                        docker build -t ${ECR_REPO_NAME2}:${DOCKER_IMAGE_TAG} .
                        docker tag ${ECR_REPO_NAME2}:${DOCKER_IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME2}:${DOCKER_IMAGE_TAG}
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME2}:${DOCKER_IMAGE_TAG}
                        '''
                    }

                    // App3 için Docker image oluşturma ve ECR'a gönderme
                    dir('react') {
                        sh '''
                        docker build -t ${ECR_REPO_NAME3}:${DOCKER_IMAGE_TAG} .
                        docker tag ${ECR_REPO_NAME3}:${DOCKER_IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME3}:${DOCKER_IMAGE_TAG}
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME3}:${DOCKER_IMAGE_TAG}
                        '''
                    }
                }
            }
        }

        stage('Get EC2 Public IP') {
            steps {
                script {
                    // EC2 instance ID'sini buluyoruz
                    def instanceId = sh(script: "aws ec2 describe-instances --filters Name=instance-state-name,Values=running --query 'Reservations[0].Instances[0].InstanceId' --output text", returnStdout: true).trim()
                    env.EC2_INSTANCE_ID = instanceId

                    // EC2 instance'ın Public IP'sini alıyoruz
                    def publicIP = sh(script: "aws ec2 describe-instances --instance-id ${EC2_INSTANCE_ID} --query 'Reservations[0].Instances[0].PublicIpAddress' --output text", returnStdout: true).trim()
                    echo "EC2 Instance Public IP: ${publicIP}"
                    
                    // Public IP'yi ortam değişkenine atıyoruz
                    env.PUBLIC_IP = publicIP
                }
            }
        }

        stage('Run Docker Compose on EC2') {
            steps {
                script {
                    // EC2'ye SSH ile bağlanıp Docker Compose'u Public IP ile çalıştırma
                    sh """
                    ssh -i /path/to/your/key.pem ec2-user@${env.PUBLIC_IP} '
                    export PUBLIC_IP=${env.PUBLIC_IP} && 
                    cd /path/to/your/docker-compose-dir &&
                    docker-compose -f docker-compose.yml up -d
                    '
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed."
        }
    }
}
