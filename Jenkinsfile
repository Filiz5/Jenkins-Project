pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = '975049995227'
        AWS_REGION = 'us-east-1'
        ECR_REPO_NAME1 = 'nodejs-image'
        ECR_REPO_NAME2 = 'postgres-image'
        ECR_REPO_NAME3 = 'react-images'
        DOCKER_IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Create Infra') {
            steps {
                script {
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
                    sh '''
                    aws ecr create-repository --repository-name ${ECR_REPO_NAME1} --region ${AWS_REGION} || echo "Repo 1 already exists"
                    aws ecr create-repository --repository-name ${ECR_REPO_NAME2} --region ${AWS_REGION} || echo "Repo 2 already exists"
                    aws ecr create-repository --repository-name ${ECR_REPO_NAME3} --region ${AWS_REGION} || echo "Repo 3 already exists"
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Build and Push Docker Images') {
            parallel {
                stage('Node.js Image') {
                    steps {
                        script {
                            dir('nodejs') {
                                sh '''
                                docker build -t ${ECR_REPO_NAME1}:${DOCKER_IMAGE_TAG} .
                                docker tag ${ECR_REPO_NAME1}:${DOCKER_IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME1}:${DOCKER_IMAGE_TAG}
                                docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME1}:${DOCKER_IMAGE_TAG}
                                '''
                            }
                        }
                    }
                }
                stage('PostgreSQL Image') {
                    steps {
                        script {
                            dir('postgresql') {
                                sh '''
                                docker build -t ${ECR_REPO_NAME2}:${DOCKER_IMAGE_TAG} .
                                docker tag ${ECR_REPO_NAME2}:${DOCKER_IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME2}:${DOCKER_IMAGE_TAG}
                                docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME2}:${DOCKER_IMAGE_TAG}
                                '''
                            }
                        }
                    }
                }
                stage('React Image') {
                    steps {
                        script {
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
            }
        }

        stage('Get EC2 Public IP') {
            steps {
                script {
                    env.EC2_PUBLIC_IP = sh(script: '''
                    aws ec2 describe-instances --filters Name=instance-state-name,Values=running --query 'Reservations[0].Instances[0].PublicIpAddress' --output text
                    ''', returnStdout: true).trim()
                }
            }
        }

        stage('Run Docker Compose on EC2') {
            steps {
                script {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i /path/to/your/key.pem ec2-user@${env.EC2_PUBLIC_IP} '
                    cd /path/to/your/docker/compose/directory &&
                    docker-compose up -d
                    '
                    """
                }
            }
        }
    }
}
