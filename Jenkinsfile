pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        IMAGE_NAME = "boobesh18/python-ci-cd"
        IMAGE_TAG  = "build-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git credentialsId: 'github-creds',
                    url: 'https://github.com/BoobeshP/python-CI-CD.git',
                    branch: 'main'
            }
        }

        stage('SonarQube Code Scan') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=python \
                        -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Build & Unit Test') {
            steps {
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install -r requirements.txt
                python -m py_compile app.py
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:$IMAGE_TAG .
                docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push $IMAGE_NAME:$IMAGE_TAG
                    docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    sed -i "s|IMAGE_TAG|$IMAGE_TAG|g" k8s/deployment.yaml

                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    kubectl rollout status deployment/python-app --timeout=120s || {
                      echo "❌ Deployment failed. Rolling back..."
                      kubectl rollout undo deployment/python-app
                      exit 1
                    }
                    '''
                }
            }
        }

        stage('Cleanup Old Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo "--- Keeping last 3 build images ---"
                    docker images "$IMAGE_NAME" --format "{{.Tag}}" \
                      | grep '^build-' \
                      | sort -V \
                      | head -n -3 \
                      | xargs -r -I {} docker rmi "$IMAGE_NAME:{}" || true
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ CI/CD Pipeline executed successfully'
        }
        failure {
            echo '❌ CI/CD Pipeline failed'
        }
    }
}
