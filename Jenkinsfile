pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-northeast-2'
        ECR_REPOSITORY = 'tier3-backend'
        VERSION_TAG = 'backend-v1.0'
        GITHUB_CREDS = credentials('github-token')
        AWS_ACCOUNT_ID = '123456789012'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.TIMESTAMP_TAG = sh(script: "date '+%Y%m%d%H%M%S'", returnStdout: true).trim()
                }
            }
        }

        stage('Test') {
            steps {
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

        stage('Get AWS Account ID') {
            steps {
                script {
                    try {
                        env.AWS_ACCOUNT_ID = sh(
                            script: 'aws sts get-caller-identity --query Account --output text',
                            returnStdout: true
                        ).trim()
                        echo "AWS 계정 ID: ${env.AWS_ACCOUNT_ID}"
                    } catch (Exception e) {
                        echo "AWS 계정 ID를 가져오는 데 실패했습니다. 기본값을 사용합니다."
                    }
                }
            }
        }

        stage('ECR Login') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
            }
        }

        stage('Build and Tag Docker Image') {
            steps {
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

        stage('Push to ECR') {
            steps {
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

        stage('Update Manifest Repository') {
            steps {
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

                sh 'chmod +x update_manifest.sh && ./update_manifest.sh'
            }
        }
    }

    post {
        always {
            script {
                try {
                    def ecrUrl = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh """
                        docker rmi ${ecrUrl}/${ECR_REPOSITORY}:${VERSION_TAG} || true
                        docker rmi ${ecrUrl}/${ECR_REPOSITORY}:latest || true
                        docker rmi ${ecrUrl}/${ECR_REPOSITORY}:${TIMESTAMP_TAG} || true
                    """
                } catch (Exception e) {
                    echo "이미지 정리 중 오류 발생: ${e.message}"
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
