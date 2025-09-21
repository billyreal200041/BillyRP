pipeline {
  agent any

  options {
    disableConcurrentBuilds()            // 禁止并发
    timeout(time: 40, unit: 'MINUTES')
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

    // 自触发判定用
    BOT_NAME   = 'jenkins-bot'
    BOT_PREFIX = 'chore(ci): bump images'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    // 里程碑：队列中较旧的构建到这会自动结束
    stage('Milestone #1') { steps { milestone(1) } }

    // 自触发保护：若最近一次提交是本 Job 回写的 bump 提交，则直接中止流水线（不视为失败）
    stage('Self-push Guard') {
      steps {
        script {
          def lastAuthor = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
          def lastMsg    = sh(script: "git log -1 --pretty=%s",  returnStdout: true).trim()
          echo "Last commit by: ${lastAuthor} | ${lastMsg}"
          if (lastAuthor == env.BOT_NAME && lastMsg.startsWith(env.BOT_PREFIX)) {
            echo "Detected bot bump commit. Aborting to avoid self-trigger loop."
            currentBuild.description = "No-op on bot commit"
            throw new hudson.AbortException("No-op build (bot commit)")
          }
        }
      }
    }

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

    // 第二个里程碑：构建前再去重
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

    // 发布前最后去重
    stage('Milestone #3') { steps { milestone(3) } }

    stage('Bump manifests & Push back to Git (main)') {
      steps {
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

            # 若没有改动就不提交、不推送，避免空推触发
            if git diff --cached --quiet; then
              echo "[INFO] No manifest changes. Skip push."
              exit 0
            fi

            git commit -m "chore(ci): bump images to ${BUILD_TAG}"

            # 统一用 https+token 推送；显式推到 main，避免 HEAD:HEAD 问题
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
    aborted { echo "Aborted (self-push guard)." }
  }
}
