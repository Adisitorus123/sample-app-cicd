pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'adiboysitorus'  
        APP_NAME = 'sample-app'
        APP_VERSION = "${BUILD_NUMBER}"
        K8S_NAMESPACE = 'default'
        GIT_REPO = 'https://github.com/Adisitorus123/sample-app-cicd.git' 
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                    userRemoteConfigs: [[
                        url: "${GIT_REPO}",
                        credentialsId: 'github-credentials'
                    ]],
                    branches: [[name: '*/main']]
                ])
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION} .
                    docker tag ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION} ${DOCKER_REGISTRY}/${APP_NAME}:latest
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-credentials', url: '']) {
                    sh """
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kube-config', variable: 'KUBECONFIG')]) {
                    sh """
                        sed -i 's|BUILD_VERSION|${APP_VERSION}|g' k8s/deployment.yaml
                        sed -i 's|DOCKER_REGISTRY|${DOCKER_REGISTRY}|g' k8s/deployment.yaml

                        kubectl apply -f k8s/deployment.yaml --kubeconfig=$KUBECONFIG
                        kubectl apply -f k8s/service.yaml --kubeconfig=$KUBECONFIG
                        kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --kubeconfig=$KUBECONFIG
                    """
                }
            }
        }
    } 

    post {
        always {
            cleanWs()
        }
        failure {
            echo 'Pipeline gagal! Periksa log untuk detail.'
        }
        success {
            echo 'Pipeline berhasil! Aplikasi sudah di-deploy ke Kubernetes.'
        }
    }
} 
