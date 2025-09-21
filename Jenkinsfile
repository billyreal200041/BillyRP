pipeline {
  agent any
  options {
    timestamps()
    timeout(time: 40, unit: 'MINUTES')
    disableConcurrentBuilds()   // 避免并发与堆积
  }

  environment {
    AWS_REGION    = 'ap-southeast-5'
    AWS_ACCOUNTID = '692859925329'
    ECR_A = "${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com/demo-web-a"
    ECR_B = "${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com/demo-web-b"
    ECR_C = "${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com/demo-web-c"
    SHORT_SHA = "${env.GIT_COMMIT?.take(7) ?: 'local'}"
    BUILD_TAG = "${SHORT_SHA}-${env.BUILD_NUMBER}"
    DOCKER_BUILDKIT = '1'
    TARGET_BRANCH = 'main'
    SKIP_BUILD = 'false'   // 动态计算
  }

  triggers { githubPush() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    // ==== 护栏：自触发/仅改 k8s 清单 的构建一律跳过 ====
    stage('Guard: skip self-triggered or k8s-only commits') {
      steps {
        script {
          def author = sh(returnStdout: true, script: "git log -1 --pretty=%ae").trim()
          def msg    = sh(returnStdout: true, script: "git log -1 --pretty=%s").trim()
          echo "Last commit author: ${author}"
          echo "Last commit msg   : ${msg}"

          // 规则 1：机器人回写或显式 [skip ci]
          def isBotCommit = (author == "jenkins-bot@local" || msg =~ /(?i)\\[skip ci]|\\[ci skip]|bump images/)

          // 规则 2：改动文件仅在 k8s/ 目录（典型自回写）
          def changed = sh(returnStdout: true, script: "git show --pretty='' --name-only HEAD").trim().split('\\n') as List
          def nonK8s = changed.findAll { !(it ==~ /^k8s\\//) && it.trim() }
          def k8sOnly = (changed && nonK8s.isEmpty())

          echo "Changed files: ${changed}"
          echo "k8s-only commit? ${k8sOnly}"

          if (isBotCommit || k8sOnly) {
            env.SKIP_BUILD = 'true'
            echo "Guard hit → skip this run."
          }
        }
      }
    }

    stage('Login ECR') {
      when { expression { env.SKIP_BUILD != 'true' } }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-access-key']]) {
          sh '''
            set -e
            echo "===> AWS identity"
            aws configure list
            aws sts get-caller-identity

            echo "===> Docker login to ECR"
            aws ecr get-login-password --region "$AWS_REGION" \
            | docker login --username AWS --password-stdin \
              ${AWS_ACCOUNTID}.dkr.ecr.${AWS_REGION}.amazonaws.com
          '''
        }
      }
    }

    stage('Build & Push Images') {
      when { expression { env.SKIP_BUILD != 'true' } }
      steps {
        sh '''
          set -e
          echo "===> Build images"
          docker build -t ${ECR_A}:latest -t ${ECR_A}:${BUILD_TAG} a
          docker build -t ${ECR_B}:latest -t ${ECR_B}:${BUILD_TAG} b
          docker build -t ${ECR_C}:latest -t ${ECR_C}:${BUILD_TAG} c

          echo "===> Push images to ECR"
          docker push ${ECR_A}:latest && docker push ${ECR_A}:${BUILD_TAG}
          docker push ${ECR_B}:latest && docker push ${ECR_B}:${BUILD_TAG}
          docker push ${ECR_C}:latest && docker push ${ECR_C}:${BUILD_TAG}
        '''
      }
    }

    stage('Bump manifests & Push back to Git') {
      when { expression { env.SKIP_BUILD != 'true' } }
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GHTOKEN')]) {
          sh '''
            set -e
            git config user.name  "jenkins-bot"
            git config user.email "jenkins-bot@local"

            update_image() {
              local file="$1" repo="$2" tag="$3"
              echo "Updating $file -> ${repo}:${tag}"
              # 采用 sed 简单可靠替换（image: 独占一行）
              sed -i -E "s|^(\\s*image:\\s*).*$|\\1${repo}:${tag}|" "$file"
            }

            update_image k8s/a/deployment.yaml ${ECR_A} ${BUILD_TAG}
            update_image k8s/b/deployment.yaml ${ECR_B} ${BUILD_TAG}
            update_image k8s/c/deployment.yaml ${ECR_C} ${BUILD_TAG}

            git add k8s/*/deployment.yaml
            git commit -m "[skip ci] chore(ci): bump images to ${BUILD_TAG}" || echo "No changes to commit."

            REPO_URL=$(git config --get remote.origin.url)
            echo "$REPO_URL" | grep -q '^http' || REPO_URL=$(echo "$REPO_URL" | sed 's#git@github.com:#https://github.com/#; s#\\.git$##').git
            REPO_URL_AUTH=$(echo "$REPO_URL" | sed "s#https://#https://${GHTOKEN}@#")

            git push "$REPO_URL_AUTH" HEAD:refs/heads/${TARGET_BRANCH}
          '''
        }
      }
    }
  }

  post {
    success {
      script {
        if (env.SKIP_BUILD == 'true') {
          echo "Skipped self-triggered/k8s-only build."
        } else {
          echo "Build ${BUILD_TAG} done. Argo CD will auto-sync shortly."
        }
      }
    }
    failure { echo "Build failed. Please check logs." }
  }
}
