pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "boobesh/python-app"
        KUBE_CONFIG = credentials('kubeconfig')
        SONARQUBE = tool 'SonarQubeScanner'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/your-repo/python-ci-cd-app'
            }
        }

        stage('Code Quality - SonarQube') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                    $SONARQUBE/bin/sonar-scanner \
                    -Dsonar.projectKey=python-app \
                    -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Build & Test') {
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
                sh 'docker build -t $DOCKER_IMAGE:latest .'
            }
        }

        stage('Docker Push') {
            steps {
                sh '''
                docker login -u user -p pass
                docker push $DOCKER_IMAGE:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml
                '''
            }
        }
    }
}
