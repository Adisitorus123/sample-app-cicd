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
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ["busybox", "sleep", "99999"]
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true
"""
            defaultContainer 'jnlp'
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
                container('node') {
                    sh 'npm ci'
                }
            }
        }

        stage('Run Tests') {
            steps {
                container('node') {
                    sh 'npm test'
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                container('kaniko') {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            mkdir -p /kaniko/.docker
                            cat > /kaniko/.docker/config.json <<EOF2
                    {"auths":{"https://index.docker.io/v1/":{"username":"${DOCKER_USER}","password":"${DOCKER_PASS}"}}}
                    EOF2
                            /kaniko/executor \
                              --context=. \
                              --dockerfile=Dockerfile \
                              --destination=${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION} \
                              --destination=${DOCKER_REGISTRY}/${APP_NAME}:latest \
                              --cache=false
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
