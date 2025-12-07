// Declarative Pipeline 시작
pipeline {
    agent any

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
                    app = docker.build("yun1code/grow-app:${env.BUILD_NUMBER}")
                }
            }
        }

        // 4) Push Docker image
        stage('Push Docker image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'yun1code') {
                        app.push()
                        app.push("latest")
                    }
                }
            }
        }

        // 5) GCP 인증 + kubeconfig 설정 (권한 문제 해결)
        stage('GCP Auth & Kubeconfig') {
            steps {
                script {
                    // gcp-sa-key: Jenkins Credential ID
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        // $WORKSPACE/.kube/config 경로 사용
                        sh '''
                            export KUBECONFIG=$WORKSPACE/.kube/config
                            mkdir -p $(dirname $KUBECONFIG)

                            # GCP 서비스 계정 인증
                            gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                            # kubeconfig 갱신 (클러스터 접속)
                            gcloud container clusters get-credentials k8s --zone asia-northeast3-a --project oss-grow
                        '''
                    }
                }
            }
        }

        // 6) Create DockerHub Pull Secret in Kubernetes
        stage('Create DockerHub Pull Secret in Kubernetes') {
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
                                  -n grow-dev \
                                  --dry-run=client -o yaml | kubectl apply -f -
                            """
                        }
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withEnv(["KUBECONFIG=$WORKSPACE/.kube/config"]) {
                        sh """
                            kubectl apply -n grow-dev -f k8s/secret.yaml
                            kubectl apply -n grow-dev -f k8s/deployment.yaml
                            kubectl apply -n grow-dev -f k8s/service.yaml
                        """
                    }
                }
            }
        }
        stage('Check rollout status') {
            steps {
                script {
                    withEnv(["KUBECONFIG=$WORKSPACE/.kube/config"]) {
                        sh "kubectl rollout status deployment/grow-app -n grow-dev"
                    }
                }
            }
        }

    } // stages 끝
} // pipeline 끝
