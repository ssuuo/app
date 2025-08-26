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
    REGISTRY   = 'harbor.harbor.svc.cluster.local'    // 내부 FQDN 통일
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
echo "[info] IMAGE   = ${REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}"
echo "[info] USER    = ${HUSER}"

# Docker config.json 생성 (auths 키 == REGISTRY 와 완전히 동일)
mkdir -p /kaniko/.docker
AUTH_B64=$(printf "%s:%s" "${HUSER}" "${HPASS}" | base64 | tr -d '\\n')
printf '{
  "auths": {
    "%s": { "auth": "%s" }
  }
}\n' "${REGISTRY}" "${AUTH_B64}" > /kaniko/.docker/config.json

echo "==== /kaniko/.docker/config.json ===="
cat /kaniko/.docker/config.json

# (옵션) 헤더 확인 (401이면 정상: 토큰 요구)
echo "[probe] HEAD https://${REGISTRY}/v2/"
/busybox/wget -S --spider "https://${REGISTRY}/v2/" 2>&1 | /busybox/head -n 4 || true

# Kaniko 실행 (self-signed → TLS 검증 스킵)
exec /kaniko/executor \
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
