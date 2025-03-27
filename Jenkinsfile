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
  - name: git
    image: alpine/git:latest
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
        AWS_REGION = 'ap-northeast-2'
        ECR_REPOSITORY = 'tier3-backend'
        VERSION_TAG = 'backend-v1.0'
        GITHUB_CREDS = credentials('github-token')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // 타임스탬프 태그 생성 (모든 컨테이너에서 사용 가능하도록)
                    env.TIMESTAMP_TAG = sh(script: "date '+%Y%m%d%H%M%S'", returnStdout: true).trim()
                }
            }
        }

        stage('Test') {
            steps {
                container('golang') {
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
                }
            }
        }

        stage('Get AWS Account ID') {
            steps {
                container('aws') {
                    script {
                        env.AWS_ACCOUNT_ID = sh(
                            script: 'aws sts get-caller-identity --query Account --output text',
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }

        stage('ECR Login') {
            steps {
                container('aws') {
                    sh "aws ecr get-login-password --region ${AWS_REGION} > /tmp/ecr_password"
                }
                container('docker') {
                    sh "cat /tmp/ecr_password | docker login --username AWS --password-stdin ${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "rm -f /tmp/ecr_password"
                }
            }
        }

        stage('Build and Tag Docker Image') {
            steps {
                container('docker') {
                    script {
                        def ecrUrl = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        
                        sh """
                            docker build -t ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} .
                            docker tag ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} ${ecrUrl}/${ECR_REPOSITORY}:latest
                            docker tag ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} ${ecrUrl}/${ECR_REPOSITORY}:${TIMESTAMP_TAG}
                        """
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                container('docker') {
                    script {
                        def ecrUrl = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        
                        sh """
                            docker push ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG}
                            docker push ${ecrUrl}/${ECR_REPOSITORY}:latest
                            docker push ${ecrUrl}/${ECR_REPOSITORY}:${TIMESTAMP_TAG}
                        """
                    }
                }
            }
        }

        stage('Update Manifest Repository') {
            steps {
                container('git') {
                    // 업데이트 스크립트 생성
                    writeFile file: 'update_manifest.sh', text: """#!/bin/sh
set -e

# Git 설정
git config --global user.name "Jenkins CI"
git config --global user.email "jenkins@example.com"

# 매니페스트 저장소 클론
git clone https://x-access-token:${GITHUB_CREDS}@github.com/Mr-Muji/3tier-manifest.git
cd 3tier-manifest

# 이미지 태그 업데이트
sed -i "s|tag: .*|tag: ${TIMESTAMP_TAG}|g" charts/backend/values.yaml

# ECR 저장소 URI 업데이트
ECR_URI="${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
sed -i "s|repository: .*|repository: \${ECR_URI}|g" charts/backend/values.yaml

# 변경사항 확인 및 커밋
if [ -n "\$(git status -s)" ]; then
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

                    // 스크립트 실행 권한 부여 및 실행
                    sh 'chmod +x update_manifest.sh && ./update_manifest.sh'
                }
            }
        }
    }

    post {
        always {
            container('docker') {
                script {
                    def ecrUrl = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    
                    sh """
                        docker rmi ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} || true
                        docker rmi ${ecrUrl}/${ECR_REPOSITORY}:latest || true
                        docker rmi ${ecrUrl}/${ECR_REPOSITORY}:${TIMESTAMP_TAG} || true
                    """
                }
            }
        }
        success {
            echo '==========================================='
            echo '빌드 성공: 이미지가 성공적으로 ECR에 푸시됨'
            echo "이미지: ${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
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
