#!groovy

def label = "jenkins"
env.DOCKERHUB_IMAGE = "devops53/hello-world"

podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-slave
  namespace: jenkins
  labels:
    component: ci
    jenkins: jenkins-agent
spec:
  serviceAccountName: jenkins
  volumes:
  - name: dind-storage
    emptyDir: {}
  containers:
  - name: git
    image: alpine/git
    command:
    - cat
    tty: true
  - name: nodejs
    image: node:latest
    command:
    - cat
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
  - name: docker
    image: docker:19.03.3-git
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_HOST
      value: tcp://docker-dind:2375
    volumeMounts:
    - mountPath: /var/lib/docker
      name: dind-storage 
  - name: helm
    image: lachlanevenson/k8s-helm:v2.16.1
    command:
    - cat
    tty: true
"""
  )

{

  node(label) {
      stage('Clone another repo') {
          checkout (
                  [ $class: 'GitSCM',
                    branches: [[name: "${env?.CHANGE_ID ? env?.GIT_COMMIT : env?.BRANCH_NAME}"]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[$class: 'CloneOption',
                                  noTags: false,
                                  reference: '/var/lib/jenkins/.git-references/apm-agent-go.git',
                                  shallow: false]],
                    submoduleCfg: [],
                    userRemoteConfigs: [[credentialsId: 'UserAndToken',
                                                         url: "https://github.com/empikls/node.is"]]]
          )



      }
  }
}

