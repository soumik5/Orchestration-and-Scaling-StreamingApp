
pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {

        AWS_ACCOUNT_ID = '025064823372'
        AWS_REGION = 'ap-south-1'

        ECR_REGISTRY = '025064823372.dkr.ecr.ap-south-1.amazonaws.com'

        ECR_PREFIX = 'streamingapp'

        MONGO_VERSION = '8.0'

    }

    stages {

        // 1. CHECKOUT SOURCE CODE

        stage('Checkout') {

            steps {

                checkout scm

                script {

                    def commitId = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    if (!commitId) {
                        error('Unable to generate Git commit ID.')
                    }

                    env.IMAGE_TAG = commitId

                    echo "Full Git Commit: ${sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()}"

                    echo "Generated Docker Image Tag: ${env.IMAGE_TAG}"

                }

            }

        }


        // 2. VERIFY ENVIRONMENT AND AWS AUTHENTICATION

        stage('Verify Environment') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'soumik-aws'
                ]]) {

                    sh '''
                        set -eu
                        set +x

                        echo "Checking Docker..."
                        docker --version

                        echo "Checking Docker Compose..."
                        docker compose version

                        echo "Checking AWS CLI..."
                        aws --version

                        echo "Configuring AWS credentials..."

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        echo "Verifying AWS authentication..."

                        ACCOUNT=$(aws sts get-caller-identity \
                            --query Account \
                            --output text)

                        echo "Authenticated AWS Account: $ACCOUNT"

                        if [ "$ACCOUNT" != "$AWS_ACCOUNT_ID" ]; then
                            echo "ERROR: AWS account mismatch!"
                            exit 1
                        fi

                        echo "AWS authentication successful!"
                    '''

                }

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

                    echo "All five application Dockerfiles found."
                '''

            }

        }


        // 4. VALIDATE ALL SIX ECR REPOSITORIES

        stage('Validate ECR Repositories') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'soumik-aws'
                ]]) {

                    sh '''
                        set -eu
                        set +x

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        for SERVICE in \
                            mongo \
                            adminservice \
                            authservice \
                            chatservice \
                            frontend \
                            streamingservice
                        do

                            aws ecr describe-repositories \
                                --repository-names "$ECR_PREFIX/$SERVICE" \
                                --region "$AWS_REGION" \
                                > /dev/null

                            echo "Repository verified: $SERVICE"

                        done

                        echo "All six ECR repositories verified successfully."
                    '''

                }

            }

        }


        // 5. LOGIN TO AWS ECR

        stage('ECR Login') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'soumik-aws'
                ]]) {

                    sh '''
                        set -eu
                        set +x

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        echo "Logging in to AWS ECR..."

                        aws ecr get-login-password \
                            --region "$AWS_REGION" |
                        docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"

                        echo "ECR login successful."
                    '''

                }

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
                            dockerfile: 'backend/adminService/Dockerfile',
                            context: 'backend'
                        ],

                        [
                            name: 'authservice',
                            dockerfile: 'backend/authService/Dockerfile',
                            context: 'backend/authService'
                        ],

                        [
                            name: 'chatservice',
                            dockerfile: 'backend/chatService/Dockerfile',
                            context: 'backend'
                        ],

                        [
                            name: 'streamingservice',
                            dockerfile: 'backend/streamingService/Dockerfile',
                            context: 'backend'
                        ]

                    ]

                    for (service in services) {

                        echo "Building ${service.name}..."

                        withEnv([
                            "SERVICE_NAME=${service.name}",
                            "DOCKERFILE_PATH=${service.dockerfile}",
                            "BUILD_CONTEXT=${service.context}"
                        ]) {

                            sh '''
                                set -eu

                                IMAGE="$ECR_REGISTRY/$ECR_PREFIX/$SERVICE_NAME:$IMAGE_TAG"

                                docker build \
                                    --pull \
                                    -f "$DOCKERFILE_PATH" \
                                    -t "$IMAGE" \
                                    "$BUILD_CONTEXT"

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

                    echo "Building frontend image..."

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


        // 9. VERIFY ALL SIX IMAGES IN ECR

        stage('Verify ECR Images') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'soumik-aws'
                ]]) {

                    sh '''
                        set -eu
                        set +x

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        for SERVICE in \
                            mongo \
                            adminservice \
                            authservice \
                            chatservice \
                            frontend \
                            streamingservice
                        do

                            echo "Verifying $SERVICE:$IMAGE_TAG..."

                            aws ecr describe-images \
                                --repository-name "$ECR_PREFIX/$SERVICE" \
                                --image-ids imageTag="$IMAGE_TAG" \
                                --region "$AWS_REGION" \
                                > /dev/null

                            echo "$SERVICE image verified successfully."

                        done

                        echo "All six images verified successfully."
                    '''

                }

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

            All six Docker images pushed to ECR.

            ===================================
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
