pipeline {
    agent any

    environment {
        REACT_APP_VERSION    = "1.0.${BUILD_ID}"
        APP_NAME             = 'learnjenkinsapp'
        AWS_DEFAULT_REGION   = 'ap-southeast-2'
        AWS_DOCKER_REGISTRY  = '143555788014.dkr.ecr.ap-southeast-2.amazonaws.com'
        AWS_ECS_CLUSTER      = 'LearnJenkinsApp-Cluster-20260316'
        AWS_ECS_SERVICE_PROD = 'LearnJenkinsApp-Service-Prod'
        AWS_ECS_TD_PROD      = 'LearnJenkinsApp-TaskDefinition-Prod'
        IMAGE_NAME           = 'myjenkinsapp'
        IMAGE_URI            = "${AWS_DOCKER_REGISTRY}/${APP_NAME}:${REACT_APP_VERSION}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build React App') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    echo "Listing workspace files..."
                    ls -la

                    echo "Node version:"
                    node --version

                    echo "NPM version:"
                    npm --version

                    echo "Installing dependencies..."
                    npm ci

                    echo "Building React app..."
                    npm run build

                    echo "Build output:"
                    ls -la
                    ls -la build
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e
                    echo "Docker version:"
                    docker version

                    echo "Building Docker image..."
                    docker build -t ${IMAGE_URI} -t ${IMAGE_NAME}:latest .

                    echo "Docker images:"
                    docker images | grep -E "${APP_NAME}|${IMAGE_NAME}" || true
                '''
            }
        }

        stage('Push Docker Image to ECR') {
            agent {
                docker {
                    image 'amazon/aws-cli:latest'
                    reuseNode true
                    args "--entrypoint='' -u root -v /var/run/docker.sock:/var/run/docker.sock"
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'my-aws',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set -e
                        echo "AWS CLI version:"
                        aws --version

                        echo "Logging in to Amazon ECR..."
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
                        docker login --username AWS --password-stdin ${AWS_DOCKER_REGISTRY}

                        echo "Pushing image to ECR..."
                        docker push ${IMAGE_URI}
                    '''
                }
            }
        }

        stage('Deploy to AWS ECS') {
            agent {
                docker {
                    image 'amazon/aws-cli:latest'
                    reuseNode true
                    args "--entrypoint=''"
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'my-aws',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set -e
                        echo "AWS CLI version:"
                        aws --version

                        echo "Registering new task definition..."
                        LATEST_TD_REVISION=$(aws ecs register-task-definition \
                          --cli-input-json file://aws/task-definition-prod.json \
                          --query 'taskDefinition.revision' \
                          --output text)

                        echo "Registered task definition revision: ${LATEST_TD_REVISION}"

                        echo "Updating ECS service..."
                        aws ecs update-service \
                          --cluster ${AWS_ECS_CLUSTER} \
                          --service ${AWS_ECS_SERVICE_PROD} \
                          --task-definition ${AWS_ECS_TD_PROD}:${LATEST_TD_REVISION}

                        echo "Waiting for ECS service to become stable..."
                        aws ecs wait services-stable \
                          --cluster ${AWS_ECS_CLUSTER} \
                          --services ${AWS_ECS_SERVICE_PROD}

                        echo "Deployment completed successfully."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed."
        }
        always {
            echo "Build finished. Cleaning up unused Docker resources..."
            sh '''
                docker image prune -f || true
            '''
        }
    }
}