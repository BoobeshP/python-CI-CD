pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        IMAGE_NAME = "boobesh18/python-ci-cd"
        IMAGE_TAG  = "build-${BUILD_NUMBER}"
        REPO_NAME  = "python-ci-cd"
        KEEP_BUILDS = "3"
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/BoobeshP/python-CI-CD.git',
                    branch: 'main'
            }
        }

        stage('Build App') {
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
                    usernameVariable: 'USER',
                    passwordVariable: 'TOKEN'
                )]) {
                    sh '''
                    echo "$TOKEN" | docker login -u "$USER" --password-stdin
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
                    kubectl rollout status deployment/python-app --timeout=120s
                    '''
                }
            }
        }

        stage('Clean Docker Hub Tags (Keep Last 3)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'TOKEN'
                )]) {
                    sh '''
                    TAGS=$(curl -s -u "$USER:$TOKEN" \
                      "https://hub.docker.com/v2/repositories/$USER/$REPO_NAME/tags/?page_size=100" \
                      | jq -r '.results[].name')

                    OLD_TAGS=$(echo "$TAGS" | grep '^build-' | sort -V | head -n -$KEEP_BUILDS || true)

                    for tag in $OLD_TAGS; do
                      echo "Deleting Docker Hub tag: $tag"
                      curl -s -X DELETE -u "$USER:$TOKEN" \
                        "https://hub.docker.com/v2/repositories/$USER/$REPO_NAME/tags/$tag/"
                    done
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
