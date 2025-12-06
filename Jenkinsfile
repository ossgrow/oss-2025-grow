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
                    app = docker.build("yun1code/grow:${env.BUILD_NUMBER}")
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

        // 5) GCP 인증 + kubeconfig 설정
        stage('GCP Auth & Kubeconfig') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh '''
                            # 서비스 계정 인증
                            gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                            # kubeconfig 갱신 (클러스터 접속)
                            gcloud container clusters get-credentials k8s \
                                --zone asia-northeast3-a \
                                --project oss-grow
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
                        withEnv(["KUBECONFIG=/var/jenkins_home/.kube/config"]) {
                            sh """
                                kubectl create secret docker-registry dockerhub-secrets \
                                  --docker-username=$DH_USER \
                                  --docker-password=$DH_TOKEN \
                                  --docker-email=none \
                                  --dry-run=client -o yaml | kubectl apply -f -
                            """
                        }
                    }
                }
            }
        }

    } // stages 끝
} // pipeline 끝
