pipeline {
  agent any

  // 防并发 + 超时 + 时间戳
  options {
    disableConcurrentBuilds()            // 禁止同一 Job 并发
    timeout(time: 40, unit: 'MINUTES')   // 整体超时
    timestamps()
  }

  // 仅由 GitHub Webhook 触发（不要再配 Poll SCM）
  triggers { githubPush() }

  environment {
    AWS_REGION    = 'ap-southeast-5'
    AWS_ACCOUNTID = '692859925329'
    ECR_A = "${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com/demo-web-a"
    ECR_B = "${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com/demo-web-b"
    ECR_C = "${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com/demo-web-c"
    SHORT_SHA = "${env.GIT_COMMIT?.take(7) ?: 'local'}"
    BUILD_TAG = "${SHORT_SHA}-${env.BUILD_NUMBER}"
    DOCKER_BUILDKIT = '1'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // 里程碑：有更新的构建排队到这儿时，旧构建自动终止
    stage('Milestone #1') { steps { milestone(1) } }

    stage('Sanity: whoami & docker') {
      steps {
        sh '''
          set -e
          echo "USER=$(whoami)"; id
          docker version >/dev/null 2>&1 || { echo "[ERROR] Docker not available for $(whoami)"; exit 1; }
        '''
      }
    }

    stage('Login ECR (AK/SK via Jenkins Credentials)') {
      steps {
        // 使用 Jenkins 全局凭据：ID=aws-access-key（类型：AWS Credentials）
        withAWS(credentials: 'aws-access-key', region: "${env.AWS_REGION}") {
          sh '''
            set -e
            echo "[INFO] AWS ID: $(aws sts get-caller-identity --query Account --output text)"
            aws ecr get-login-password --region "$AWS_REGION" \
            | docker login --username AWS --password-stdin \
              ${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com
          '''
        }
      }
    }

    // 再次去重，防止旧构建继续往下覆盖新提交
    stage('Milestone #2') { steps { milestone(2) } }

    stage('Build & Push Images') {
      steps {
        sh '''
          set -e
          echo "[Build] Tag=${BUILD_TAG}"

          docker build -t ${ECR_A}:latest -t ${ECR_A}:${BUILD_TAG} a
          docker build -t ${ECR_B}:latest -t ${ECR_B}:${BUILD_TAG} b
          docker build -t ${ECR_C}:latest -t ${ECR_C}:${BUILD_TAG} c

          docker push ${ECR_A}:latest && docker push ${ECR_A}:${BUILD_TAG}
          docker push ${ECR_B}:latest && docker push ${ECR_B}:${BUILD_TAG}
          docker push ${ECR_C}:latest && docker push ${ECR_C}:${BUILD_TAG}
        '''
      }
    }

    // 发布前最后的去重
    stage('Milestone #3') { steps { milestone(3) } }

    stage('Bump manifests & Push back to Git (main)') {
      steps {
        // GitHub Token（Secret text），ID=github-token（至少 repo 权限）
        withCredentials([string(credentialsId: 'github-token', variable: 'GHTOKEN')]) {
          sh '''
            set -e
            git config user.name  "jenkins-bot"
            git config user.email "jenkins-bot@local"

            # 优先 yq；失败则回退 sed
            if ! command -v yq >/dev/null 2>&1; then
              (sudo apt-get update -y && sudo apt-get install -y yq) || true
            fi

            update_image() {
              local file="$1" repo="$2" tag="$3"
              if command -v yq >/dev/null 2>&1; then
                yq -i ".spec.template.spec.containers[0].image = \\"${repo}:${tag}\\"" "$file"
              else
                # sed 回退：要求 image: 独占一行
                sed -i -E "s|^(\\s*image:\\s*).*$|\\1${repo}:${tag}|" "$file"
              fi
            }

            update_image k8s/a/deployment.yaml ${ECR_A} ${BUILD_TAG}
            update_image k8s/b/deployment.yaml ${ECR_B} ${BUILD_TAG}
            update_image k8s/c/deployment.yaml ${ECR_C} ${BUILD_TAG}

            git add k8s/*/deployment.yaml
            git commit -m "chore(ci): bump images to ${BUILD_TAG}" || echo "[INFO] No changes to commit."

            # 统一用 https + token 推送；显式推送到 main（避免 HEAD:HEAD 问题）
            REPO_URL=$(git config --get remote.origin.url)
            echo "$REPO_URL" | grep -q '^http' || REPO_URL=$(echo "$REPO_URL" | sed 's#git@github.com:#https://github.com/#; s#\\.git$##').git
            REPO_URL_AUTH=$(echo "$REPO_URL" | sed "s#https://#https://${GHTOKEN}@#")

            git push "$REPO_URL_AUTH" HEAD:refs/heads/main
          '''
        }
      }
    }
  }

  post {
    success { echo "Build ${BUILD_TAG} done. Argo CD will auto-sync shortly." }
    failure { echo "Build failed. Please check logs." }
  }
}
