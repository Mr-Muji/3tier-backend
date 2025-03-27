pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: golang
    image: golang:1.19
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: aws
    image: amazon/aws-cli:latest
    command:
    - cat
    tty: true
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }

    environment {
        AWS_REGION = 'ap-northeast-2' // AWS 리전 설정
        ECR_REPOSITORY = 'tier3-backend' // ECR 저장소 이름 설정
        VERSION_TAG = 'backend-v1.0' // 버전 태그
        // 타임스탬프 태그 형식: YYYYMMDDHHmmss (년월일시분초)
        TIMESTAMP_TAG = sh(script: "date '+%Y%m%d%H%M%S'", returnStdout: true).trim()
        AWS_ACCOUNT_ID = sh(
            script: 'aws sts get-caller-identity --query Account --output text',
            returnStdout: true
        ).trim()
    }

    stages {
        stage('Checkout') {
            steps {
                // 소스 코드 체크아웃
                checkout scm
                echo '코드 체크아웃 완료'
            }
        }

        stage('Test') {
            steps {
                // Go 환경 설정 및 테스트 실행
                sh '''
                    echo "Go 버전 확인"
                    go version
                    echo "Go 모듈 다운로드"
                    go mod download
                    echo "Go 린트 도구 설치"
                    go get golang.org/x/lint/golint
                    echo "코드 검증 실행"
                    make validate
                '''
                echo '테스트 완료'
            }
        }

        stage('ECR Login') {
            steps {
                // ECR 로그인
                script {
                    echo 'ECR 로그인 시작'
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin \
                        ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    """
                    echo 'ECR 로그인 완료'
                }
            }
        }

        stage('Build and Tag Docker Image') {
            steps {
                script {
                    def ecrUrl = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                    echo 'Docker 이미지 빌드 시작'
                    // 이미지를 빌드하고 태그 추가
                    sh """
                        docker build -t ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} .
                        docker tag ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} \
                            ${ecrUrl}/${ECR_REPOSITORY}:latest
                        docker tag ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} \
                            ${ecrUrl}/${ECR_REPOSITORY}:${TIMESTAMP_TAG}
                    """
                    echo "Docker 이미지 빌드 및 태그 추가 완료 (태그: ${VERSION_TAG}, latest, ${TIMESTAMP_TAG})"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    def ecrUrl = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                    echo 'ECR에 이미지 푸시 시작'
                    // 모든 태그 푸시
                    sh """
                        docker push ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG}
                        docker push ${ecrUrl}/${ECR_REPOSITORY}:latest
                        docker push ${ecrUrl}/${ECR_REPOSITORY}:${TIMESTAMP_TAG}
                    """
                    echo "이미지 푸시 완료 (태그: ${VERSION_TAG}, latest, ${TIMESTAMP_TAG})"
                }
            }
        }

        stage('Update Manifest Repository') {
            steps {
                script {
                    echo '매니페스트 저장소 업데이트 시작'

                    // Jenkins에 등록된 GitHub 자격 증명 사용
                    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                        // Git 사용자 설정
                        sh """
                            git config --global user.name "Jenkins CI"
                            git config --global user.email "jenkins@example.com"

                            # 매니페스트 저장소 클론
                            git clone https://x-access-token:${GITHUB_TOKEN}@github.com/Mr-Muji/3tier-manifest.git
                            cd 3tier-manifest

                            # 이미지 태그 업데이트 (타임스탬프 태그 사용)
                            sed -i "s|tag: .*|tag: ${TIMESTAMP_TAG}|g" charts/backend/values.yaml

                            # ECR 저장소 URI 업데이트
                            ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
                            sed -i "s|repository: .*|repository: \${ECR_URI}|g" charts/backend/values.yaml

                            # 변경사항 확인
                            git status

                            # 변경사항이 있는 경우에만 커밋 및 푸시
                            if [[ -n \$(git status -s) ]]; then
                                git add charts/backend/values.yaml
                                git commit -m "Update backend image to ${TIMESTAMP_TAG}"
                                git push
                                echo "매니페스트 저장소 업데이트 완료: 이미지 태그가 ${TIMESTAMP_TAG}로 변경되었습니다."
                            else
                                echo "변경사항이 없습니다. 매니페스트 저장소 업데이트를 건너뜁니다."
                            fi

                            # 정리
                            cd ..
                            rm -rf 3tier-manifest
                        """
                    }
                    echo '매니페스트 저장소 업데이트 완료'
                }
            }
        }
    }

    post {
        always {
            script {
                def ecrUrl = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                echo '로컬 Docker 이미지 정리'
                sh """
                    docker rmi ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} || true
                    docker rmi ${ecrUrl}/${ECR_REPOSITORY}:latest || true
                    docker rmi ${ecrUrl}/${ECR_REPOSITORY}:${TIMESTAMP_TAG} || true
                """
            }
        }
        success {
            echo '==========================================='
            echo '빌드 성공: 이미지가 성공적으로 ECR에 푸시됨'
            echo "이미지: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
            echo "태그: ${VERSION_TAG}, latest, ${TIMESTAMP_TAG}"
            echo '==========================================='
        }
        failure {
            echo '==========================================='
            echo '빌드 실패: 파이프라인 실행 중 오류 발생'
            echo '==========================================='
        }
    }
}
