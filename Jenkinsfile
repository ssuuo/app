pipeline {
  agent {
    kubernetes {
      label 'kaniko-build'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2-debug
    tty: true
    command: ["/busybox/busybox","sh","-c","tail -f /dev/null"]
  - name: tools
    image: alpine:3.20
    tty: true
    command: ["/bin/sh","-c","tail -f /dev/null"]
"""
    }
  }

  environment {
    REGISTRY   = 'harbor.harbor.svc.cluster.local'
    IMAGE_REPO = 'project/myapp'
    IMAGE_TAG  = "${BUILD_NUMBER}-${env.GIT_COMMIT?.take(7) ?: 'manual'}"
  }

  stages {
    stage('Build & Push to Harbor') {
      steps {
        container('kaniko') {
          withCredentials([usernamePassword(credentialsId: 'harbor-robot', usernameVariable: 'HUSER', passwordVariable: 'HPASS')]) {
            sh '''#!/busybox/sh
set -eu

echo "[info] REGISTRY=${REGISTRY}"
echo "[info] IMAGE = ${REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}"
echo "[info] USER  = ${HUSER}"


mkdir -p /kaniko/.docker
AUTH_B64=$(printf "%s:%s" "${HUSER}" "${HPASS}" | base64 | tr -d '\\n')
printf '{
  "auths": {
    "%s": { "auth": "%s" }
  }
}\n' "${REGISTRY}" "${AUTH_B64}" > /kaniko/.docker/config.json

echo "==== /kaniko/.docker/config.json ===="
cat /kaniko/.docker/config.json


echo "[probe] HEAD https://${REGISTRY}/v2/"
/busybox/wget -S --spider "https://${REGISTRY}/v2/" 2>&1 | /busybox/head -n 4 || true

/kaniko/executor \
  --dockerfile Dockerfile \
  --context "${PWD}" \
  --destination "${REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}" \
  --cache=true \
  --verbosity=debug \
  --skip-tls-verify
'''
          }
        }
      }
    }
  }
}

    stage('Update GitOps Repository') {
      steps {
        container('tools') {
          withCredentials([usernamePassword(credentialsId: 'token', usernameVariable: 'GUSER', passwordVariable: 'GTOKEN')]) {
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