pipeline {
    agent any

    environment {
        REACT_APP_VERSION    = "1.0.${BUILD_ID}"
        APP_NAME             = 'learnjenkinsapp'
        AWS_DEFAULT_REGION   = 'ap-southeast-2'
        AWS_DOCKER_REGISTRY = '143555788014.dkr.ecr.ap-southeast-2.amazonaws.com'
        AWS_ECS_CLUSTER      = 'LearnJenkinsApp-Cluster-20260316'
        AWS_ECS_SERVICE_PROD = 'LearnJenkinsApp-Service-Prod'
        AWS_ECS_TD_PROD      = 'LearnJenkinsApp-TaskDefinition-Prod'
        IMAGE_NAME           = 'myjenkinsapp'
    }

    stages {

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Build Docker image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'my-aws',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set -e
                        docker version
                        docker build -t $AWS_DOCKER_REGISTRY/${APP_NAME}:${REACT_APP_VERSION} -t ${IMAGE_NAME}:latest .
                        aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_DOCKER_REGISTRY 
                        docker push $AWS_DOCKER_REGISTRY/${APP_NAME}:${REACT_APP_VERSION} -t ${IMAGE_NAME}:latest 
                    '''
                }
            }
        }

        stage('Deploy to AWS') {
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
                        aws --version

                        LATEST_TD_REVISION=$(aws ecs register-task-definition \
                          --cli-input-json file://aws/task-definition-prod.json \
                          --query 'taskDefinition.revision' \
                          --output text)

                        echo "Registered task definition revision: ${LATEST_TD_REVISION}"

                        aws ecs update-service \
                          --cluster ${AWS_ECS_CLUSTER} \
                          --service ${AWS_ECS_SERVICE_PROD} \
                          --task-definition ${AWS_ECS_TD_PROD}:${LATEST_TD_REVISION}

                        aws ecs wait services-stable \
                          --cluster ${AWS_ECS_CLUSTER} \
                          --services ${AWS_ECS_SERVICE_PROD}
                    '''
                }
            }
        }
    }
}