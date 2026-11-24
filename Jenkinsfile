
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'vinee0311/kubernetes-sample-app'
        IMAGE_TAG = 'latest'
        K8S_NAMESPACE = 'jenkins-poc'
        K8S_SERVER = 'https://192.168.167.197:6443'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Node.js Application') {
            steps {
                sh '''
                    echo "Checking application files..."
                    test -f app.js
                    test -f package.json
                    test -f Dockerfile
                    test -f nodejs-deployment.yaml
                    test -f nodejs-service.yaml
                    echo "All application files found."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                          -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'k8s-jenkins-token',
                        variable: 'K8S_TOKEN'
                    ),
                    string(
                        credentialsId: 'k8s-ca-cert',
                        variable: 'K8S_CA'
                    )
                ]) {
                    sh '''
                        mkdir -p "$WORKSPACE/.kube"

                        echo "$K8S_CA" | base64 -d \
                          > "$WORKSPACE/.kube/ca.crt"

                        kubectl config set-cluster poc-cluster \
                          --server="$K8S_SERVER" \
                          --certificate-authority="$WORKSPACE/.kube/ca.crt"

                        kubectl config set-credentials jenkins-poc \
                          --token="$K8S_TOKEN"

                        kubectl config set-context poc-context \
                          --cluster=poc-cluster \
                          --user=jenkins-poc \
                          --namespace="$K8S_NAMESPACE"

                        kubectl config use-context poc-context

                        kubectl apply -f nodejs-deployment.yaml
                        kubectl apply -f nodejs-service.yaml

                        kubectl rollout status deployment/nodejs-sample-app \
                          -n "$K8S_NAMESPACE" --timeout=180s

                        kubectl get pods -n "$K8S_NAMESPACE"
                        kubectl get services -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully!'
        }
        failure {
            echo 'CI/CD pipeline failed. Check the console output.'
        }
        always {
            sh 'rm -rf "$WORKSPACE/.kube"'
        }
    }
}
