pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '678364258357'
        AWS_REGION     = 'us-east-1'
        CLUSTER_NAME   = 'devops-eks-cluster' // <-- שנה לשם הקלאסטר שלך כשתקים את ה-EKS
        
        REGISTRY_URL   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_BACKEND  = "task-backend"
        IMAGE_FRONTEND = "task-frontend"
        
        VERSION        = "v1.0.${env.BUILD_NUMBER}"
    }

    stages {
        stage('Setup Tools') {
            steps {
                echo 'Installing AWS CLI, Docker client, and kubectl tools inside Jenkins...'
                sh '''
                    apt-get update && apt-get install -y unzip curl
                    
                    # התקנת AWS CLI
                    if ! command -v aws &> /dev/null; then
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip -o awscliv2.zip
                        ./aws/install --update
                    fi
                    
                    # התקנת Docker CLI
                    if ! command -v docker &> /dev/null; then
                        curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-24.0.7.tgz -o docker.tgz
                        tar xzvf docker.tgz
                        cp docker/docker /usr/local/bin/
                    fi
                    
                    # התקנת kubectl
                    if ! command -v kubectl &> /dev/null; then
                        curl -LO "https://dl.k8s.io/release/\$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mv kubectl /usr/local/bin/
                    fi
                    
                    aws --version
                    docker --version
                    kubectl version --client
                '''
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('AWS ECR Login') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY_URL}"
            }
        }

        stage('Build Docker Images') {
            steps {
                echo "Building version ${VERSION}..."
                sh "docker build -t ${IMAGE_BACKEND}:${VERSION} ./backend"
                sh "docker build -t ${IMAGE_FRONTEND}:${VERSION} ./frontend"
            }
        }

        stage('Tag & Push to ECR') {
            steps {
                sh "docker tag ${IMAGE_BACKEND}:${VERSION} ${REGISTRY_URL}/${IMAGE_BACKEND}:${VERSION}"
                sh "docker tag ${IMAGE_FRONTEND}:${VERSION} ${REGISTRY_URL}/${IMAGE_FRONTEND}:${VERSION}"
                
                sh "docker push ${REGISTRY_URL}/${IMAGE_BACKEND}:${VERSION}"
                sh "docker push ${REGISTRY_URL}/${IMAGE_FRONTEND}:${VERSION}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        echo "Attempting to connect to EKS Cluster: ${CLUSTER_NAME}..."
                        // עדכון ה-kubeconfig של ג'נקינס מול AWS
                        sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
                        
                        echo "Updating Kubernetes Cluster with version ${VERSION}..."
                        sh "kubectl apply -f ./k8s/"
                        sh "kubectl set image deployment/backend-deployment backend=${REGISTRY_URL}/${IMAGE_BACKEND}:${VERSION}"
                        sh "kubectl set image deployment/frontend-deployment frontend=${REGISTRY_URL}/${IMAGE_FRONTEND}:${VERSION}"
                    } catch (Exception e) {
                        echo "--------------------------------------------------------"
                        echo "WARNING: Could not deploy to EKS (${e.getMessage()})."
                        echo "This is expected if EKS is not running yet or during local testing."
                        echo "Images were successfully pushed to ECR. Skipping deployment stage."
                        echo "--------------------------------------------------------"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline finished successfully! Version ${VERSION} processing complete."
        }
        failure {
            echo "Pipeline failed. Check the logs above."
        }
    }
}