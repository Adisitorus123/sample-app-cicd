pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:3383.vc8881d4b_0e76-1
    args: ["\$(JENKINS_SECRET)", "\$(JENKINS_NAME)"]
  - name: node
    image: node:18-alpine
    command: ["cat"]
    tty: true
  - name: docker
    image: docker:24.0-cli
    command: ["cat"]
    tty: true
    env:
    - name: DOCKER_HOST
      value: tcp://localhost:2375
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
            defaultContainer 'node'
        }
    }

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
                container('docker') {
                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION} .
                        docker tag ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION} ${DOCKER_REGISTRY}/${APP_NAME}:latest
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withDockerRegistry([credentialsId: 'docker-hub-credentials', url: '']) {
                        sh """
                            docker push ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}
                            docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kube-config', variable: 'KUBECONFIG')]) {
                        sh """
                            sed -i 's|BUILD_VERSION|${APP_VERSION}|g' k8s/deployment.yaml
                            sed -i 's|DOCKER_REGISTRY|${DOCKER_REGISTRY}|g' k8s/deployment.yaml
                            kubectl apply -f k8s/deployment.yaml
                            kubectl apply -f k8s/service.yaml
                            kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            deleteDir()
        }
        failure {
            echo 'Pipeline gagal! Periksa log untuk detail.'
        }
        success {
            echo 'Pipeline berhasil! Aplikasi sudah di-deploy ke Kubernetes.'
        }
    }
}
