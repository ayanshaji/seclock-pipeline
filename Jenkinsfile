pipeline {

    agent any

    environment {
        AWS_REGION = "ap-south-1"
        AWS_ACCOUNT_ID = "024757002695"

        ECR_REPOSITORY = "seclock"
        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE = "${ECR_REGISTRY}/${ECR_REPOSITORY}"

        GIT_REPO_NAME = "seclock-pipeline"
        GIT_USER_NAME = "ayanshaji"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate

                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest test_e2e.py
                '''
            }
        }

        stage('Create ECR Repository') {
            steps {
                sh '''
                    echo "Checking ECR repository..."

                    aws ecr describe-repositories \
                        --repository-names ${ECR_REPOSITORY} \
                        --region ${AWS_REGION} \
                    || \
                    aws ecr create-repository \
                        --repository-name ${ECR_REPOSITORY} \
                        --region ${AWS_REGION}
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build -t ${ECR_IMAGE}:${IMAGE_TAG} .

                    docker tag \
                        ${ECR_IMAGE}:${IMAGE_TAG} \
                        ${ECR_IMAGE}:latest
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    echo "Logging into ECR..."

                    aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    echo "Pushing image to ECR..."

                    docker push ${ECR_IMAGE}:${IMAGE_TAG}
                    docker push ${ECR_IMAGE}:latest
                '''
            }
        }

        stage('Update Kubernetes Deployment') {
            steps {
                sh '''
                    echo "Updating Kubernetes deployment..."

                    sed -i "s|image: .*|image: ${ECR_IMAGE}:${IMAGE_TAG}|g" k8s/deployment.yml

                    echo "Updated image:"
                    grep "image:" k8s/deployment.yml
                '''
            }
        }

        stage('Commit and Push to GitHub') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'git-hub',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {

                    sh '''
                        echo "Committing Kubernetes manifest..."

                        git config user.email "jenkins@localhost"
                        git config user.name "Jenkins"

                        git add k8s/deployment.yml

                        git commit \
                            -m "Update seclock image to ${IMAGE_TAG} [skip ci]" \
                            || echo "No changes to commit"

                        echo "Pushing updated deployment.yml to GitHub..."

                        git push \
                            https://${GITHUB_TOKEN}@github.com/${GITHUB_USERNAME}/${GIT_REPO_NAME}.git \
                            HEAD:main
                    '''
                }
            }
        }
    }

    post {

        always {
            sh 'rm -rf venv'
        }

        success {
            echo "Pipeline completed successfully!"
        }

        failure {
            echo "Pipeline failed!"
        }
    }
}
