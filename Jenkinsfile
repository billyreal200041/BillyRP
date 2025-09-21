pipeline {
  agent any

  // 防并发 + 超时
  options {
    disableConcurrentBuilds()            // 禁止同一 Job 并发
    timeout(time: 40, unit: 'MINUTES')   // 整体超时
    timestamps()
  }

  // 仅由 GitHub Webhook 触发
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

    // 里程碑：如果有更新的构建排到这里，老的在此处自动结束
    stage('Milestone #1') {
      steps {
        milestone(1)
      }
    }

    stage('Sanity: whoami & docker') {
      steps {
        sh '''
          set -e
          echo "USER=$(whoami)"; id
          docker version >/dev/null 2>&1 || { echo "Docker not available for $(whoami)"; exit 1; }
        '''
      }
    }

    stage('Login ECR (AK/SK)') {
      steps {
        // 使用 Jenkins 中 ID=aws-access-key 的 AWS 凭据 + 固定 region
        withAWS(credentials: 'aws-access-key', region: "${env.AWS_REGION}") {
          sh '''
            set -e
            echo "AWS ID: $(aws sts get-caller-identity --query Account --output text)"
            aws ecr get-login-password --region "$AWS_REGION" \
            | docker login --username AWS --password-stdin \
              ${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com
          '''
        }
      }
    }

    // 里程碑：构建前再次去重；更“新的”提交来了会让老构建止步于此
    stage('Milestone #2') {
      steps {
        milestone(2)
      }
    }

    stage('Build & Push Images') {
      steps {
        // 构建与推送不需要再次调用 AWS CLI，因此不强制包 withAWS
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

    // 里程碑：防止较旧构建在发布阶段覆盖较新的提交
    stage('Milestone #3') {
      steps {
        milestone(3)
      }
    }

    stage('Bump manifests & Push back to Git') {
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GHTOKEN')]) {
          sh '''
            set -e
            git config user.name  "jenkins-bot"
            git config user.email "jenkins-bot@local"

            # 优先 yq；失败则用 sed 回退
            if ! command -v yq >/dev/null 2>&1; then
              (sudo apt-get update -y && sudo apt-get install -y yq) || true
            fi

            update_image() {
              local file="$1" repo="$2" tag="$3"
              if command -v yq >/dev/null 2>&1; then
                yq -i ".spec.template.spec.containers[0].image = \\"${repo}:${tag}\\"" "$file"
              else
                sed -i -E "s|^(\\s*image:\\s*).*$|\\1${repo}:${tag}|" "$file"
              fi
            }

            update_image k8s/a/deployment.yaml ${ECR_A} ${BUILD_TAG}
            update_image k8s/b/deployment.yaml ${ECR_B} ${BUILD_TAG}
            update_image k8s/c/deployment.yaml ${ECR_C} ${BUILD_TAG}

            git add k8s/*/deployment.yaml
            git commit -m "chore(ci): bump images to ${BUILD_TAG}" || echo "No changes to commit."

            REPO_URL=$(git config --get remote.origin.url)
            echo "$REPO_URL" | grep -q '^http' || REPO_URL=$(echo "$REPO_URL" | sed 's#git@github.com:#https://github.com/#; s#\\.git$##').git
            REPO_URL_AUTH=$(echo "$REPO_URL" | sed "s#https://#https://${GHTOKEN}@#")

            # 推回同一分支
            git push "$REPO_URL_AUTH" HEAD:$(git rev-parse --abbrev-ref HEAD)
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
