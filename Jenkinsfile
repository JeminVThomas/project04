
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '994878981749'
        ECR_REPOSITORY = 'project04'
        ECR_REGISTRY = '994878981749.dkr.ecr.ap-south-1.amazonaws.com'

        // Replace with your actual GitOps repository URL
        GITOPS_REPO = 'https://github.com/JeminVThomas/project04-gitops.git'
        GITOPS_BRANCH = 'main'
        GITOPS_MANIFEST = 'k8s/deployment.yaml'
    }

    stages {

        stage('Checkout') {
    steps {
        checkout scm

        script {
            env.IMAGE_TAG = env.GIT_COMMIT.take(7)

            env.IMAGE_URI =
                "${ECR_REGISTRY}/${ECR_REPOSITORY}:${env.IMAGE_TAG}"
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
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=project04 \
                              -Dsonar.projectName=Project04 \
                              -Dsonar.sources=. \
                              -Dsonar.exclusions=.venv/**,venv/**,**/__pycache__/**,media/**,reports/**
                        '''
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
                dir('project4') {
                    sh '''
                        docker build -t ${IMAGE_URI} .
                    '''
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region ${AWS_REGION} |
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}

                    docker push ${IMAGE_URI}
                    docker logout ${ECR_REGISTRY}
                '''
            }
        }

        stage('Update GitOps Repository') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'github-gitops-token',
                        variable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        AUTH_HEADER="AUTHORIZATION: basic $(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 -w0)"

                        rm -rf gitops-repo

                        git -c http.extraheader="$AUTH_HEADER" \
                            clone --branch ${GITOPS_BRANCH} \
                            ${GITOPS_REPO} gitops-repo

                        cd gitops-repo

                        if [ ! -f "${GITOPS_MANIFEST}" ]; then
                            echo "GitOps deployment manifest not found"
                            exit 1
                        fi

                        sed -i -E \
                          "s|^[[:space:]]*image:.*|          image: ${IMAGE_URI}|" \
                          "${GITOPS_MANIFEST}"

                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        if ! git diff --quiet; then
                            git add "${GITOPS_MANIFEST}"
                            git commit -m "Update Project04 image to ${IMAGE_TAG}"

                            git -c http.extraheader="$AUTH_HEADER" push origin ${GITOPS_BRANCH}
                        else
                            echo "No GitOps manifest changes detected"
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI pipeline successful. GitOps repository updated.'
            echo 'Argo CD will reconcile the updated deployment manifest.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage logs.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}

