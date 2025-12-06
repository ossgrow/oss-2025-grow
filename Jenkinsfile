// Declarative Pipeline 시작
pipeline {

    // Jenkins가 어떤 agent(노드)에서 빌드를 실행할지 지정
    // 'any'는 Jenkins가 가능한 아무 노드나 선택해 실행한다는 의미
    agent any

    // environment 블록: 파이프라인 전체에서 사용할 환경변수 정의
    // 지금은 사용하지 않지만 DB 관련 env 변수를 Jenkins Credential로
    // 불러올 때 사용 가능 (필요할 때만 활성화하면 됨)
    /*environment {
        // DB_HOST = credentials('db_host')   // Jenkins Credential에서 불러오기
        // DB_USER = credentials('db_user')
        // DB_PASSWORD = credentials('db_password')
        // DB_DATABASE = credentials('db_database')
        // DB_PORT = credentials('db_port')
    }*/

    // 전체 빌드 과정을 단계별로 구분하는 영역
    stages {

        // 1) Checkout 단계: GitHub에서 코드 가져오기
        stage('Checkout') {
            steps {
                // checkout scm은 Multibranch Pipeline에서
                // 자동으로 해당 브랜치의 코드를 clone하는 명령
                checkout scm
            }
        }

        // 2) Install dependencies 단계: npm install 실행
        stage('Install dependencies') {
            steps {
                script {
                    // Jenkins 컨테이너에는 Node.js가 설치되어 있지 않기 때문에
                    // node:18 Docker 이미지를 사용해서 npm install 실행
                    docker.image('node:18').inside {
                        // Node.js 패키지 설치
                        sh 'npm install'
                    }
                }
            }
        }

        // 3) Build Docker image: Docker 이미지 빌드
        stage('Build Docker image') {
            steps {
                script {
                    // 현재 프로젝트 코드를 기반으로 Docker 이미지 생성
                    // 이미지 이름: yun1code/grow
                    // 태그: Jenkins의 빌드 번호(BUILD_NUMBER)
                    app = docker.build("yun1code/grow:${env.BUILD_NUMBER}")
                }
            }
        }

        // 4) Push Docker image: DockerHub로 이미지 push
        stage('Push Docker image') {
            steps {
                script {
                    // Jenkins Credential ID 'yun1code'를 credentialsId로 사용하여
                    // DockerHub에 로그인한다.
                    docker.withRegistry('https://registry.hub.docker.com', 'yun1code') {
                        // 빌드 번호 태그로 push
                        app.push()
                        // latest 태그로 push (Kubernetes deployment가 최신 이미지 pull 가능)
                        app.push("latest")
                    }
                }
            }
        }

        // 5) Create DockerHub Pull Secret in Kubernetes
        // Kubernetes 클러스터에서 DockerHub의 private 이미지를 pull하려면
        // docker-registry 타입의 secret이 필요하다.
        // Jenkins가 DockerHub ID/Token을 사용하여 자동 생성해주는 단계.
        stage('Create DockerHub Pull Secret in Kubernetes') {
            steps {
                script {

                    // Jenkins Credential(ID: 'yun1code')에서
                    // DockerHub username/password(Access Token)를 읽어와
                    // DH_USER, DH_TOKEN 환경변수에 저장한다.
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'yun1code',      // Jenkins Credential ID
                            usernameVariable: 'DH_USER',     // DockerHub ID
                            passwordVariable: 'DH_TOKEN'     // DockerHub Token
                        )
                    ]) {
                        withEnv(["KUBECONFIG=/var/jenkins_home/.kube/config"]) {
                        // kubectl 명령으로 K8s secret 생성
                        // dry-run=client: 실제 파일 생성 없이 yaml 출력
                        // 그 출력 결과를 그대로 kubectl apply로 적용
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

 }// pipeline 끝
