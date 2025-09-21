pipeline {
  agent any
  options {
    timestamps()
    timeout(time: 40, unit: 'MINUTES')
    disableConcurrentBuilds()            // 不并发
    skipDefaultCheckout(false)
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
  }

  triggers { githubPush() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    // === 强护栏：命中就立刻终止整条流水线（不再执行任何 stage） ===
    stage('Guard (hard stop)') {
      steps {
        script {
          def author    = sh(returnStdout: true, script: "git log -1 --pretty=%ae").trim()
          def msg       = sh(returnStdout: true, script: "git log -1 --pretty=%s").trim()
          def msgLC     = msg.toLowerCase()
          def changed   = sh(returnStdout: true, script: "git show --pretty= --name-only HEAD").trim().split('\\n').collect{ it.trim() }.findAll{ it }
          def k8sOnly   = (changed && changed.every { it.startsWith('k8s/') })

          echo "Last commit author: ${author}"
          echo "Last commit msg   : ${msg}"
          echo "Changed files     : ${changed}"
          echo "k8s-only commit?  : ${k8sOnly}"

          def isBotAuthor = (author == 'jenkins-bot@local')
          def isSkipMsg   = (msgLC.contains('[skip ci]') || msgLC.contains('[ci skip]') || msgLC.contains('chore(ci): bump images'))

          if (isBotAuthor || isSkipMsg || k8sOnly) {
            echo "Guard HIT → abort pipeline (self-commit / [skip ci] / k8s-only)."
            currentBuild.displayName = "#${env.BUILD_NUMBER} [SKIPPED]"
            // 关键：抛出 AbortException 结束整条流水线（灰色 ABORTED，不是失败）
            throw new hudson.AbortException('Skipped by guard')
          }
        }
      }
    }

    stage('Login ECR') {
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
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GHTOKEN')]) {
          sh '''
            set -e
            git config user.name  "jenkins-bot"
            git config user.email "jenkins-bot@local"

            update_image() {
              local file="$1" repo="$2" tag="$3"
              echo "Updating $file -> ${repo}:${tag}"
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
