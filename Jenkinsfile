pipeline {
  agent {
    kubernetes {
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    tty: true
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2
    tty: true
    command: ["/busybox/busybox","sh","-c","tail -f /dev/null"]
"""
    }
  }
  stages {
    stage('Smoke') {
      steps {
        container('jnlp') {
          sh 'echo hello && uname -a'
        }
      }
    }
    stage('Build & Push') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context "${WORKSPACE}" \
              --dockerfile Dockerfile \
              --destination "harbor.local:30443/project/myapp:${BUILD_NUMBER}-${GIT_COMMIT:0:7}" \
              --cache=true
          '''
        }
      }
    }
  }
}
