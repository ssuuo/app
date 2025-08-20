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
    REGISTRY_PUSH = "harbor-registry.harbor.svc.cluster.local:5000"  // HTTP
    // 배포용(외부) - 클러스터 노드/파드가 이미지 풀 때 사용할 호스트
    REGISTRY_PULL = "harbor.127.0.0.1.nip.io"                        // 너가 쓰던 Ingress 호스트

    PROJECT  = "project"
    IMAGE    = "myapp"
    TAG      = "${env.BUILD_NUMBER}"

    // Kaniko가 실제로 푸시할 대상
    DEST     = "${REGISTRY_PUSH}/${PROJECT}/${IMAGE}:${TAG}"

    // GitOps values.yaml 에 기록할 repository (외부 풀용)
    REPO_PULL = "${REGISTRY_PULL}/${PROJECT}/${IMAGE}"
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
                "${REGISTRY_PUSH}" "${HUSER}" "${HPASS}" > /kaniko/.docker/config.json
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

              REPO_PULL="${REPO_PULL}" TAG="${TAG}" /usr/local/bin/yq -i '
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
}
