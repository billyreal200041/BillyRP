pipeline {
  agent any
  options {
    timestamps()
    timeout(time: 40, unit: 'MINUTES')
    disableConcurrentBuilds()
    skipDefaultCheckout(false)
    // 如果想在 Guard 标红时自动停止后续 stage，也可以打开这行+用 unstable()
    // skipStagesAfterUnstable()
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
    SKIP_BUILD = 'false'     // Guard 会把它改成 'true'
  }

  triggers { githubPush() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Guard (hard stop)') {
      steps {
        script {
          // 读取最后一次提交信息
          def author  = sh(returnStdout: true, script: "git log -1 --pretty=%ae").trim()
          def msg     = sh(returnStdout: true, script: "git log -1 --pretty=%s").trim()
          def msgLC   = msg.toLowerCase()
          def changed = sh(returnStdout: true, script: "git show --pretty= --name-only HEAD").trim()
                           .split('\\n').collect{ it.trim() }.findAll{ it }
          def k8sOnly = (changed && changed.every { it.startsWith('k8s/') })

          echo "Last commit author: ${author}"
          echo "Last commit msg   : ${msg}"
          echo "Changed files     : ${changed}"
          echo "k8s-only commit?  : ${k8sOnly}"

          def isBotAuthor = (author == 'jenkins-bot@local')
          def isSkipMsg   = (msgLC.contains('[skip ci]') || msgLC.contains('[ci skip]') || msgLC.contains('chore(ci): bump images'))

          if (isBotAuthor || isSkipMsg || k8sOnly) {
            echo "Guard HIT → mark build ABORTED & skip remaining stages."
            currentBuild.displayName = "#${env.BUILD_NUMBER} [SKIPPED]"
            env.SKIP_BUILD = 'true'

            // 标记当前阶段/构建为 ABORTED（不会触发脚本安全限制）
            catchError(buildResult: 'ABORTED', stageResult: 'ABORTED') {
              error('Skipped by guard') // 被 catchError 捕获，不会让流水线失败
            }
          }
        }
      }
    }

    stage('Login ECR') {
      when { expression { return env.SKIP_BUILD != 'true' } }
      steps {
        // 如果你用 AK/SK：确保 Jenkins 有 ID=aws-access-key 的 AWS Credentials
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
      when { expression { return env.SKIP_BUILD != 'true' } }
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
      when { expression { return env.SKIP_BUILD != 'true' } }
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GHTOKEN')]) {
          sh '''
            set -e
            git config user.name  "jenkins-bot"
            git config user.email "jenkins-bot@local"

            update_image() {
              local file="$1" repo="$2" tag="$3"
              echo "Updating $file -> ${repo}:${tag}"
              # 你的 YAML image: 行独占一行，sed 即可
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
    success { echo "Done." }
    aborted { echo "Aborted by Guard (expected for k8s-only / [skip ci] commits)." }
    failure { echo "Build failed. Please check logs." }
  }
}
