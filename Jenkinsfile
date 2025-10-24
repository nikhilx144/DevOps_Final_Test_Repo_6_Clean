pipeline {
    agent any

    environment {
        IMAGE_NAME = "college-website-prac1"
        ECR_REPO   = "661979762009.dkr.ecr.ap-south-2.amazonaws.com/devops_ci_cd_final_prac_6_clean"
        REGION     = "ap-south-2"
    }

    stages {
        stage('Clone Repository') {
            steps {
                echo '📦 Cloning repository... '
                git branch: 'master', url: 'https://github.com/nikhilx144/DevOps_Final_Test_Repo_6_Clean.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '🐳 Building Docker image...'
                sh '''
                    docker build -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Push to AWS ECR') {
            steps {
                echo '🚀 Pushing image to AWS ECR...'
                withCredentials([usernamePassword(credentialsId: 'aws-username-pass-access-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

                        aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REPO}

                        docker tag ${IMAGE_NAME}:latest ${ECR_REPO}:latest
                        docker push ${ECR_REPO}:latest
                    '''
                }
            }
        }

        stage('Deploy with Terraform') {
            steps {
                echo '🏗️ Deploying EC2 instance and running Docker container...'
                withCredentials([usernamePassword(credentialsId: 'aws-username-pass-access-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    dir('Terraform') {
                        sh '''
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

                            terraform init -reconfigure
                            terraform apply -auto-approve
                        '''
                    }
                }
            }
        }

        stage('Deploy Prometheus EC2') {
            steps {
                echo '📊 Deploying Prometheus EC2 instance...'
                withCredentials([usernamePassword(credentialsId: 'aws-username-pass-access-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    dir('Terraform') {
                        sh '''
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

                            terraform init -reconfigure
                            terraform apply -auto-approve -target=aws_instance.prometheus
                        '''
                    }
                }
            }
        }

        stage('Setup Prometheus Server') {
            steps {
                echo '⚙️ Setting up Prometheus EC2 and configuration...'
                withCredentials([
                    usernamePassword(credentialsId: 'aws-username-pass-access-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
                    sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'KEY_FILE')
                ]) {
                    dir('Terraform') {
                        sh '''
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

                            PROMETHEUS_IP=$(terraform output -raw prometheus_public_ip)
                            echo "📂 Copying Prometheus configuration..."
                            scp -i $KEY_FILE -o StrictHostKeyChecking=no prometheus/prometheus.yml ec2-user@$PROMETHEUS_IP:/home/ec2-user/

                            echo "🚀 Starting Prometheus container..."
                            ssh -i $KEY_FILE -o StrictHostKeyChecking=no ec2-user@$PROMETHEUS_IP bash << EOF
                                # Create config directory
                                sudo mkdir -p /etc/prometheus

                                # Move the config file and fix permissions
                                sudo mv /home/ec2-user/prometheus.yml /etc/prometheus/prometheus.yml
                                sudo chown -R ec2-user:ec2-user /etc/prometheus
                                sudo chmod 644 /etc/prometheus/prometheus.yml

                                # Remove any old container
                                sudo docker rm -f prometheus || true

                                # Run Prometheus container
                                sudo docker run -d --name prometheus -p 9090:9090 \\
                                    -v /etc/prometheus:/etc/prometheus \\
                                    prom/prometheus --config.file=/etc/prometheus/prometheus.yml
                            EOF
                        '''
                    }
                }
            }
        }

    }

    post {
        success {
            echo '✅ Docker image pushed, EC2 deployed, and website is running!'
            echo '🎉 Open the site in your browser using the EC2 Public IP or DNS.'
        }
        failure {
            echo '❌ Build or deployment failed!'
        }
    }
}
