
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '994878981749'
        ECR_REPOSITORY = 'project04'

        IMAGE_URI = '994878981749.dkr.ecr.ap-south-1.amazonaws.com/project04'

        SONAR_SERVER = 'sonarqube'
        SONAR_SCANNER = 'SonarScanner'
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    env.IMAGE_TAG = "${env.GIT_COMMIT.take(7)}-${env.BUILD_NUMBER}"
                    echo "Building image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('project4') {
                    sh '''
                        python3 -m venv .venv
                        . .venv/bin/activate

                        pip install --upgrade pip
                        pip install -r requirements.txt
                    '''
                }
            }
        }

        stage('Django Validation') {
            steps {
                dir('project4') {
                    sh '''
                        . .venv/bin/activate
                        python manage.py check
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('project4') {
                    script {
                        def scannerHome = tool "${SONAR_SCANNER}"

                        withSonarQubeEnv("${SONAR_SERVER}") {
                            sh """
                                "${scannerHome}/bin/sonar-scanner" \
                                -Dsonar.projectKey=project04 \
                                -Dsonar.projectName=Project04 \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=.venv/**,venv/**,**/__pycache__/**,media/**,reports/**,**/*.sql,**/*.sqlite3
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                        -t ${IMAGE_URI}:${IMAGE_TAG} \
                        project4
                '''
            }
        }

        
stage('Push Image to ECR') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'aws-ecr-credentials',
                usernameVariable: 'AWS_ACCESS_KEY_ID',
                passwordVariable: 'AWS_SECRET_ACCESS_KEY'
            )
        ]) {
            sh '''
                set -e

                export AWS_DEFAULT_REGION=ap-south-1

                echo "Logging in to Amazon ECR..."

                aws ecr get-login-password --region ap-south-1 |
                docker login \
                    --username AWS \
                    --password-stdin 994878981749.dkr.ecr.ap-south-1.amazonaws.com

                echo "Pushing Docker image..."

                docker push 994878981749.dkr.ecr.ap-south-1.amazonaws.com/project04:${IMAGE_TAG}

                echo "Image pushed successfully."
            '''
        }
    }
}

        stage('Update Kubernetes Manifest') {
            steps {
                script {
                    sh '''
                        set -e

                        if [ ! -f k8s/deployment.yaml ]; then
                            echo "ERROR: k8s/deployment.yaml not found"
                            exit 1
                        fi

                        sed -i -E \
                          "s|(^[[:space:]]*image:[[:space:]]*).*|\\1${IMAGE_URI}:${IMAGE_TAG}|" \
                          k8s/deployment.yaml

                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        git add k8s/deployment.yaml

                        if git diff --cached --quiet; then
                            echo "Kubernetes manifest already contains this image."
                        else
                            git commit -m "Update project04 image to ${IMAGE_TAG}"

                            echo "Kubernetes manifest updated."
                        fi
                    '''

                    withCredentials([
                        string(
                            credentialsId: 'github',
                            variable: 'GITHUB_TOKEN'
                        )
                    ]) {
                        sh '''
                            set +x

                            AUTH=$(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 -w0)

                            git -c http.extraheader="AUTHORIZATION: basic $AUTH" \
                                push origin HEAD:main
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Docker image: ${IMAGE_URI}:${IMAGE_TAG}"
        }

        failure {
            echo "Pipeline failed. Check the failed stage logs."
        }

        always {
            echo "Pipeline execution finished."
        }
    }
}
