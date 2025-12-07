pipeline {
    agent any

    environment {
        IMAGE_NAME = "yun1code/grow-app"
        IMAGE_TAG = "${env.BUILD_NUMBER}"   // 매 빌드마다 고유 태그 생성
        K8S_NAMESPACE = "default"
    }

    stages {

        // 1) Checkout
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 2) Install dependencies
        stage('Install dependencies') {
            steps {
                script {
                    docker.image('node:18').inside {
                        sh 'npm install'
                    }
                }
            }
        }

        // 3) Build Docker image
        stage('Build Docker image') {
            steps {
                script {
                    app = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        // 4) Push Docker image
        stage('Push Docker image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'yun1code') {
                        app.push()               // ex) grow-app:22
                        app.push("latest")       // latest 유지
                    }
                }
            }
        }

        // 5) GCP Auth & Kubeconfig
        stage('GCP Auth & Kubeconfig') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh '''
                            export KUBECONFIG=$WORKSPACE/.kube/config
                            mkdir -p $(dirname $KUBECONFIG)

                            gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                            gcloud container clusters get-credentials k8s --zone asia-northeast3-a --project oss-grow
                        '''
                    }
                }
            }
        }

        // 6) Create DockerHub Pull Secret
        stage('Create DockerHub Pull Secret') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'yun1code',
                            usernameVariable: 'DH_USER',
                            passwordVariable: 'DH_TOKEN'
                        )
                    ]) {
                        withEnv(["KUBECONFIG=$WORKSPACE/.kube/config"]) {
                            sh """
                                kubectl create secret docker-registry dockerhub-secrets \
                                  --docker-username=$DH_USER \
                                  --docker-password=$DH_TOKEN \
                                  --docker-email=none \
                                  -n ${K8S_NAMESPACE} \
                                  --dry-run=client -o yaml | kubectl apply -f -
                            """
                        }
                    }
                }
            }
        }

        // 7) 🚀 Auto Rolling Update to Kubernetes
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withEnv(["KUBECONFIG=$WORKSPACE/.kube/config"]) {
                        sh """
                            kubectl set image deployment/grow-app \
                              grow-app=${IMAGE_NAME}:${IMAGE_TAG} \
                              -n ${K8S_NAMESPACE} \
                              --record
                        """
                    }
                }
            }
        }

        // 8) Check rollout success
        stage('Check rollout status') {
            steps {
                script {
                    withEnv(["KUBECONFIG=$WORKSPACE/.kube/config"]) {
                        sh "kubectl rollout status deployment/grow-app -n ${K8S_NAMESPACE}"
                    }
                }
            }
        }
    }
}
