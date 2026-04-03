pipeline {
    agent any

    options {
        disableConcurrentBuilds()   // ✅ only one pipeline at a time
        timestamps()
    }

    environment {
        IMAGE_NAME = "boobesh18/python-ci-cd"
        IMAGE_TAG  = "build-${BUILD_NUMBER}"
        REPO_NAME  = "python-ci-cd"
        KEEP_BUILDS = "3"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/BoobeshP/python-CI-CD.git',
                    branch: 'main'
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
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                    echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
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

        stage('Clean Old Docker Hub Tags') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                    echo "🧹 Cleaning old Docker Hub tags (keep last $KEEP_BUILDS)..."

                    TAGS=$(curl -s -u "$DOCKER_USER:$DOCKER_TOKEN" \
                      "https://hub.docker.com/v2/repositories/$DOCKER_USER/$REPO_NAME/tags/?page_size=100" \
                      | jq -r '.results[].name')

                    DELETE_TAGS=$(echo "$TAGS" \
                      | grep '^build-' \
                      | sort -V \
                      | head -n -$KEEP_BUILDS || true)

                    for tag in $DELETE_TAGS; do
                      echo "Deleting remote tag: $tag"
                      curl -s -X DELETE -u "$DOCKER_USER:$DOCKER_TOKEN" \
                        "https://hub.docker.com/v2/repositories/$DOCKER_USER/$REPO_NAME/tags/$tag/"
                    done

                    echo "✅ Docker Hub cleanup complete"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ CI/CD Pipeline SUCCESS'
        }
        failure {
            echo '❌ CI/CD Pipeline FAILED'
        }
    }
}
