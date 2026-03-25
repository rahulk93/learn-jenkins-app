pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        REACT_APP_VERSION = "1.0.${BUILD_ID}"
        APP_NAME = 'learnjenkinsapp'
        AWS_DEFAULT_REGION = 'ap-southeast-2'
        AWS_DOCKER_REGISTRY = '143555788014.dkr.ecr.ap-southeast-2.amazonaws.com'
        AWS_ECS_CLUSTER = 'LearnJenkinsApp-Cluster-20260316'
        AWS_ECS_SERVICE_PROD = 'LearnJenkinsApp-Service-Prod'
        AWS_ECS_TD_PROD = 'LearnJenkinsApp-TaskDefinition-Prod'
        IMAGE_URI = "${AWS_DOCKER_REGISTRY}/${APP_NAME}:${REACT_APP_VERSION}"
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
                    set -eux
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    test -d build
                '''
            }
        }

        stage('Build and Push Docker Image') {
            agent {
                docker {
                    image 'docker:26-cli'
                    reuseNode true
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'my-aws',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set -eux

                        apk add --no-cache python3 py3-pip py3-virtualenv bash

                        python3 -m venv /tmp/awsenv
                        . /tmp/awsenv/bin/activate
                        pip install --no-cache-dir --upgrade pip
                        pip install --no-cache-dir awscli

                        aws --version
                        docker version

                        aws ecr get-login-password --region "$AWS_DEFAULT_REGION" \
                          | docker login --username AWS --password-stdin "$AWS_DOCKER_REGISTRY"

                        docker build -t "$IMAGE_URI" .
                        docker push "$IMAGE_URI"
                    '''
                }
            }
        }

        stage('Render Task Definition') {
            steps {
                sh '''
                    set -eux
                    cp aws/task-definition-prod.json aws/task-definition-rendered.json
                    sed -i "s|__IMAGE__|$IMAGE_URI|g" aws/task-definition-rendered.json
                    cat aws/task-definition-rendered.json
                '''
            }
        }

        stage('Deploy to AWS ECS') {
            agent {
                docker {
                    image 'amazon/aws-cli:2.34.16'
                    reuseNode true
                    args '--entrypoint=""'
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'my-aws',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set -eux

                        LATEST_TD_REVISION=$(aws ecs register-task-definition \
                        --cli-input-json file://aws/task-definition-rendered.json \
                        --query 'taskDefinition.revision' \
                        --output text)

                        aws ecs update-service \
                        --cluster "$AWS_ECS_CLUSTER" \
                        --service "$AWS_ECS_SERVICE_PROD" \
                        --task-definition "$AWS_ECS_TD_PROD:$LATEST_TD_REVISION"

                        if ! aws ecs wait services-stable \
                        --cluster "$AWS_ECS_CLUSTER" \
                        --services "$AWS_ECS_SERVICE_PROD"; then

                        echo "Service did not stabilize. Dumping diagnostics..."

                        aws ecs describe-services \
                            --cluster "$AWS_ECS_CLUSTER" \
                            --services "$AWS_ECS_SERVICE_PROD"

                        TASKS=$(aws ecs list-tasks \
                            --cluster "$AWS_ECS_CLUSTER" \
                            --service-name "$AWS_ECS_SERVICE_PROD" \
                            --query 'taskArns' \
                            --output text || true)

                        if [ -n "$TASKS" ]; then
                            aws ecs describe-tasks \
                            --cluster "$AWS_ECS_CLUSTER" \
                            --tasks $TASKS
                        fi

                        exit 1
                        fi
                    '''
                }
            }
        }
    }
}