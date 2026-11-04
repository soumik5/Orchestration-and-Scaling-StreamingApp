pipeline {

    agent {
        label 'any'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {

        AWS_ACCOUNT_ID = '025064823372'
        AWS_REGION = 'ap-south-1'

        ECR_REGISTRY =
            '025064823372.dkr.ecr.ap-south-1.amazonaws.com'

        ECR_PREFIX = 'streamingapp'

        MONGO_VERSION = '8.0'

        IMAGE_TAG = ''
    }

    stages {

        // 1. CHECKOUT SOURCE CODE

        stage('Checkout') {

            steps {

                checkout scm

                script {

                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                }

                echo "Image tag: ${IMAGE_TAG}"

            }
        }


        // 2. VERIFY ENVIRONMENT

        stage('Verify Environment') {

            steps {

                sh '''
                    set -eu

                    docker --version
                    aws --version

                    ACCOUNT=$(aws sts get-caller-identity \
                        --query Account \
                        --output text)

                    if [ "$ACCOUNT" != "$AWS_ACCOUNT_ID" ]; then
                        echo "Incorrect AWS account!"
                        exit 1
                    fi

                    echo "AWS authentication successful."
                '''

            }
        }


        // 3. VALIDATE DOCKERFILES

        stage('Validate Dockerfiles') {

            steps {

                sh '''
                    set -eu

                    echo "Checking application Dockerfiles..."

                    test -f backend/adminService/Dockerfile
                    test -f backend/authService/Dockerfile
                    test -f backend/chatService/Dockerfile
                    test -f backend/streamingService/Dockerfile
                    test -f frontend/Dockerfile

                    echo "All five Dockerfiles found."
                '''

            }
        }


        // 4. VERIFY ECR REPOSITORIES

        stage('Validate ECR Repositories') {

            steps {

                sh '''
                    set -eu

                    for SERVICE in \
                        mongo \
                        adminservice \
                        authservice \
                        chatservice \
                        frontend \
                        streamingservice
                    do

                        aws ecr describe-repositories \
                            --repository-names \
                            "$ECR_PREFIX/$SERVICE" \
                            --region "$AWS_REGION" \
                            > /dev/null

                        echo "Repository verified: $SERVICE"

                    done
                '''

            }
        }


        // 5. LOGIN TO AWS ECR

        stage('ECR Login') {

            steps {

                sh '''
                    set -eu

                    aws ecr get-login-password \
                        --region "$AWS_REGION" |
                    docker login \
                        --username AWS \
                        --password-stdin "$ECR_REGISTRY"

                    echo "ECR login successful."
                '''

            }
        }


        // 6. PULL AND PUSH OFFICIAL MONGODB IMAGE

        stage('Build and Push MongoDB') {

            steps {

                sh '''
                    set -eu

                    echo "Pulling official MongoDB image..."

                    docker pull "mongo:$MONGO_VERSION"

                    TARGET="$ECR_REGISTRY/$ECR_PREFIX/mongo"

                    docker tag \
                        "mongo:$MONGO_VERSION" \
                        "$TARGET:$IMAGE_TAG"

                    docker push "$TARGET:$IMAGE_TAG"

                    echo "MongoDB image pushed successfully."
                '''

            }
        }


        // 7. BUILD AND PUSH FOUR BACKEND SERVICES

        stage('Build and Push Backend Services') {

            steps {

                script {

                    def services = [
                        [
                            name: 'adminservice',
                            path: 'backend/adminService'
                        ],
                        [
                            name: 'authservice',
                            path: 'backend/authService'
                        ],
                        [
                            name: 'chatservice',
                            path: 'backend/chatService'
                        ],
                        [
                            name: 'streamingservice',
                            path: 'backend/streamingService'
                        ]
                    ]

                    for (service in services) {

                        echo "Building ${service.name}"

                        withEnv([
                            "SERVICE_NAME=${service.name}",
                            "BUILD_PATH=${service.path}"
                        ]) {

                            sh '''
                                set -eu

                                IMAGE="$ECR_REGISTRY/$ECR_PREFIX/$SERVICE_NAME:$IMAGE_TAG"

                                docker build \
                                    --pull \
                                    -f "$BUILD_PATH/Dockerfile" \
                                    -t "$IMAGE" \
                                    "$BUILD_PATH"

                                docker push "$IMAGE"

                                echo "Pushed: $IMAGE"
                            '''

                        }

                    }

                }

            }
        }


        // 8. BUILD AND PUSH FRONTEND

        stage('Build and Push Frontend') {

            steps {

                sh '''
                    set -eu

                    IMAGE="$ECR_REGISTRY/$ECR_PREFIX/frontend:$IMAGE_TAG"

                    docker build \
                        --pull \
                        -f frontend/Dockerfile \
                        -t "$IMAGE" \
                        frontend

                    docker push "$IMAGE"

                    echo "Frontend image pushed successfully."
                '''

            }
        }


        // 9. VERIFY ALL SIX IMAGES

        stage('Verify ECR Images') {

            steps {

                sh '''
                    set -eu

                    for SERVICE in \
                        mongo \
                        adminservice \
                        authservice \
                        chatservice \
                        frontend \
                        streamingservice
                    do

                        echo "Verifying $SERVICE..."

                        aws ecr describe-images \
                            --repository-name "$ECR_PREFIX/$SERVICE" \
                            --image-ids imageTag="$IMAGE_TAG" \
                            --region "$AWS_REGION" \
                            > /dev/null

                        echo "$SERVICE image verified."

                    done

                    echo "All six images verified successfully."
                '''

            }
        }

    }


    // POST BUILD ACTIONS

    post {

        success {

            echo """
            ===================================
            PIPELINE SUCCESS
            ===================================

            AWS Account : ${AWS_ACCOUNT_ID}
            AWS Region  : ${AWS_REGION}
            Image Tag   : ${IMAGE_TAG}

            All six images pushed to ECR.
            """

        }

        failure {

            echo "Pipeline failed. Check Jenkins console output."

        }

        always {

            sh '''
                docker logout "$ECR_REGISTRY" || true
            '''

        }

    }

}

