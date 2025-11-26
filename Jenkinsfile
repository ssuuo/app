pipeline {
  agent {
    kubernetes {
      label 'kaniko-build'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  hostAliases:
  - ip: "172.18.0.4" 
    hostnames:
    - "harbor.local"
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2-debug
    tty: true
    command:
    - /busybox/busybox
    - sh
    - -c
    - tail -f /dev/null
    volumeMounts:
    - name: kaniko-cache
      mountPath: /kaniko/cache
  - name: tools
    image: alpine:3.20
    tty: true
    command:
    - /bin/sh
    - -c
    - tail -f /dev/null
  volumes:
  - name: kaniko-cache
    emptyDir: {}
"""
    }
  }

  environment {
    REGISTRY      = 'harbor.local:30443'
    IMAGE_REPO    = 'testproject/myapp'
    GITOPS_REPO   = 'https://github.com/ssuuo/git.git'
    GITOPS_BRANCH = 'main'
    VALUES_FILE   = 'charts/myapp/values.yaml'
    REPO_PULL    = "${REGISTRY}/${IMAGE_REPO}"
  }

  stages {
    stage('Checkout App') {
      steps {
        git branch: 'main', url: 'https://github.com/ssuuo/app.git'
        script {
          env.SHORT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.SHORT_SHA}"
          env.TAG       = env.IMAGE_TAG
          env.DEST      = "${env.REPO_PULL}:${env.IMAGE_TAG}"
          echo "Using IMAGE_TAG=${env.IMAGE_TAG}"
        }
      }
    }
    
    stage('Build & Push to Harbor') {
      steps {
        container('kaniko') {
          withCredentials([usernamePassword(credentialsId: 'testrobot', usernameVariable: 'HUSER', passwordVariable: 'HPASS')]) {
            sh '''
              set -eux

              mkdir -p /kaniko/.docker

              # base64 auth (username:password)
              AUTH_B64=$(printf "%s:%s" "${HUSER}" "${HPASS}" | base64 | tr -d '\n')

              # registry + token 서버 자격증명
              printf '{
                "auths": {
                  "%s": {
                  "auth": "%s"
                 }
                }
              }\n' "$REGISTRY" "$AUTH_B64" > /kaniko/.docker/config.json

              cat /kaniko/.docker/config.json

              # Kaniko 실행
              /kaniko/executor \\
                --dockerfile Dockerfile \\
                --context ${PWD} \\
                --destination ${REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG} \\
                --cache=true \\
                --verbosity=debug \\
                --skip-tls-verify \\
            '''
          }
        }
      }
    }

    stage('Update GitOps Repository') {
      steps {
        container('tools') {
          withCredentials([usernamePassword(credentialsId: 'gittoken', usernameVariable: 'GUSER', passwordVariable: 'GTOKEN')]) {
            sh '''
              set -eux
              apk add --no-cache git curl
              # yq 설치
              curl -L https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 \
                -o /usr/local/bin/yq
              chmod +x /usr/local/bin/yq

              rm -rf gitops && mkdir gitops && cd gitops
              git config --global user.name "jenkins-bot"
              git config --global user.email "jenkins-bot@local"

              GUSER="${GUSER:-x-access-token}"
              git clone "https://${GUSER}:${GTOKEN}@${GITOPS_REPO#https://}" .
              git checkout ${GITOPS_BRANCH}

              test -f "${VALUES_FILE}"

              REPO_PULL="${REPO_PULL}" TAG="${TAG}" yq -i '
                .image.repository = strenv(REPO_PULL) |
                .image.tag        = strenv(TAG)
              ' "${VALUES_FILE}"

              git add ${VALUES_FILE}
              git commit -m "chore: update image tag to ${TAG} (${DEST})" || echo "no changes"
              git push origin ${GITOPS_BRANCH}
            '''
          }
        }
      }
    }
  }

  post {
    always {
      echo 'Pipeline completed!'
    }
    success {
      echo "✅ Successfully built and pushed ${env.DEST}"
    }
    failure {
      echo "❌ Pipeline failed!"
    }
  }
}