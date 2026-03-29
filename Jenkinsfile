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
        IMAGE_URI            = "${AWS_DOCKER_REGISTRY}/${APP_NAME}:${REACT_APP_VERSION}"
    }

    stages {

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
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la build
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e
                    docker version
                    docker build -t ${IMAGE_URI} .
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'my-aws',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set -e
                        aws --version
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} \
                          | docker login --username AWS --password-stdin ${AWS_DOCKER_REGISTRY}
                        docker push ${IMAGE_URI}
                    '''
                }
            }
        }

        stage('Deploy to AWS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'my-aws',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set -e
                        aws --version
                        jq --version

                        cp aws/task-definition-prod.json aws/task-definition-prod-rendered.json

                        sed -i "s|#APP_VERSION#|${REACT_APP_VERSION}|g" aws/task-definition-prod-rendered.json

                        LATEST_TD_ARN=$(aws ecs register-task-definition \
                          --cli-input-json file://aws/task-definition-prod-rendered.json \
                          --query 'taskDefinition.taskDefinitionArn' \
                          --output text)

                        aws ecs update-service \
                          --cluster ${AWS_ECS_CLUSTER} \
                          --service ${AWS_ECS_SERVICE_PROD} \
                          --task-definition ${LATEST_TD_ARN}

                        aws ecs wait services-stable \
                          --cluster ${AWS_ECS_CLUSTER} \
                          --services ${AWS_ECS_SERVICE_PROD}
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}