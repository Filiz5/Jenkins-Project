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
                    // Terraform key ve altyapı oluşturma
                    sh '''
                    terraform init
                    terraform apply -auto-approve
                    '''
                }
            }
        }

        stage('Wait for Image Push') {
            steps {
                script {
                    // Docker image'ların ECR'a itilmesinden sonra bekleme
                    sleep time: 30, unit: 'SECONDS'
                }
            }
        }

        stage('Wait for EC2 3/3 Status Checks') {
            steps {
                script {
                    // EC2 instance ID'yi al (output olarak main.tf dosyasından instance id'yi almalısın)
                    def instanceId = sh(script: "terraform output -raw instance_id", returnStdout: true).trim()

                    // AWS CLI kullanarak status check kontrolü
                    def statusCheckPassed = false
                    while (!statusCheckPassed) {
                        def status = sh(script: "aws ec2 describe-instance-status --instance-id ${instanceId} --query 'InstanceStatuses[0].InstanceStatus.Status' --output text", returnStdout: true).trim()

                        if (status == "ok") {
                            statusCheckPassed = true
                        } else {
                            echo "EC2 instance is not ready yet. Waiting for 30 seconds..."
                            sleep(time: 30, unit: "SECONDS")
                        }
                    }
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

        stage('Build and Push App 1 Docker Image') {
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
                }
            }
        }

        stage('Build and Push App 2 Docker Image') {
            steps {
                script {
                    // App2 için Docker image oluşturma ve ECR'a gönderme
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

        stage('Build and Push App 3 Docker Image') {
            steps {
                script {
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

        stage('Deploy App with Ansible') {
            steps {
                script {
                    // Ansible playbook kullanarak deploy
                    sh '''
                    ansible-playbook -i /home/ec2-user/inventory_aws_ec2.yml playbook.yaml
                    '''
                }
            }
        }
    }
}
