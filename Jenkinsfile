pipeline {
  agent any
  options {
    timestamps()
    timeout(time: 40, unit: 'MINUTES')
    disableConcurrentBuilds()
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
    SKIP_BUILD = 'false'
  }

  triggers { githubPush() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Guard (soft stop, no error)') {
      steps {
        script {
          def author  = sh(returnStdout: true, script: "git log -1 --pretty=%ae").trim()
          def msg     = sh(returnStdout: true, script: "git log -1 --pretty=%s").trim().toLowerCase()
          def changed = sh(returnStdout: true, script: "git show --pretty= --name-only HEAD").trim()
                           .split('\\n').collect{ it.trim() }.findAll{ it }
          def k8sOnly = (changed && changed.every { it.startsWith('k8s/') })

          echo "Last commit author: ${author}"
          echo "Last commit msg   : ${msg}"
          echo "Changed files     : ${changed}"
          echo "k8s-only commit?  : ${k8sOnly}"

          def isBotAuthor = (author == 'jenkins-bot@local')
          def isSkipMsg   = (msg.contains('[skip ci]') || msg.contains('[ci skip]') || msg.contains('chore(ci): bump images'))

          if (isBotAuthor || isSkipMsg || k8sOnly) {
            echo "Guard HIT → mark build ABORTED & skip remaining stages."
            currentBuild.displayName = "#${env.BUILD_NUMBER} [SKIPPED]"
            currentBuild.result = 'ABORTED'
            env.SKIP_BUILD = 'true'
            return   // 软退出本 stage，不抛异常
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
            aws sts get-caller-identity
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
          docker build -t ${ECR_A}:latest -t ${ECR_A}:${BUILD_TAG} a
          docker build -t ${ECR_B}:latest -t ${ECR_B}:${BUILD_TAG} b
          docker build -t ${ECR_C}:latest -t ${ECR_C}:${BUILD_TAG} c
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
    aborted { echo "Aborted by Guard (expected self-commit / [skip ci] / k8s-only)." }
    failure { echo "Build failed. Please check logs." }
  }
}
