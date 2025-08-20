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
    REGISTRY = "harbor.127.0.0.1.nip.io"
    PROJECT  = "project"
    IMAGE    = "myapp"
    TAG      = "${env.BUILD_NUMBER}"
    DEST     = "${REGISTRY}/${PROJECT}/${IMAGE}:${TAG}"

    GITOPS_REPO   = "https://github.com/ssuuo/git.git"
    GITOPS_BRANCH = "main"
    VALUES_FILE   = "charts/myapp/values.yaml"
  }

  stages {
    stage('Checkout App') {
      steps {
        // app-repo에서 Jenkinsfile 실행 중이라면 생략 가능
        // 멀티브랜치가 아니면 명시:
        // git branch: 'main', url: 'https://github.com/yourorg/app-repo.git'
        echo "Using workspace of app-repo"
      }
    }

    stage('Build & Push (Kaniko)') {
      steps {
        container('kaniko') {
          withCredentials([usernamePassword(credentialsId: 'harbor-robot', usernameVariable: 'HUSER', passwordVariable: 'HPASS')]) {
            sh '''
              set -eux
              echo "PWD=$PWD"; ls -la
              test -f Dockerfile

              mkdir -p /kaniko/.docker
              printf '{"auths":{"%s":{"username":"%s","password":"%s"}}}\n' \
                "${REGISTRY}" "${HUSER}" "${HPASS}" > /kaniko/.docker/config.json
              cat /kaniko/.docker/config.json

              /kaniko/executor \
                --dockerfile Dockerfile \
                --context "${WORKSPACE}" \
                --destination "${DEST}" \
                --cache=true \
                --skip-tls-verify
            '''
          }
        }
      }
    }

    stage('Update GitOps Tag & Push') {
      steps {
        container('tools') {
          withCredentials([usernamePassword(credentialsId: 'usernamepat', usernameVariable: 'GUSER', passwordVariable: 'GTOKEN')]) {
            sh '''
              set -eux
              apk add --no-cache git curl
              # yq 설치(선택)
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

              TAG="${TAG}" /usr/local/bin/yq -i '.image.tag = strenv(TAG)' "${VALUES_FILE}"

              git add ${VALUES_FILE}
              git commit -m "chore: update image tag to ${TAG} (${DEST})" || echo "no changes"
              git push origin ${GITOPS_BRANCH}
            '''
          }
        }
      }
    }
  }
}
