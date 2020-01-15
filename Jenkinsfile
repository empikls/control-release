#!groovy


def label = "jenkins"


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
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
  - name: helm
    image: lachlanevenson/k8s-helm:v2.16.1
    command:
    - cat
    tty: true
"""
  )

        {

            node(label) {

                stage('Clone config repo') {
                    checkout scm
                    echo "tag from Job1 : ${params.tagFromJob1}"
                }
                def branchName = params.tagFromJob1
                if (ischangeSetList()) {
                    branchName = "${values.image.tag}"
                }
                stage('Checkout App repo') {
                    checkout([$class           : 'GitSCM',
                              branches         : [[name: branchName]],
                              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                }

                if (isMaster()) {
                    stage('Deploy DEV release') {
                        confValues = "./dev/values.yaml"
                        appName = app-dev
                        nameSpace = dev
                        dockerTag = params.tagFromJob1
                        deploy(confValues, appName, nameSpace, dockerTag)
                    }
                }
                if (isBuildingTag()) {
                    stage('Deploy QA release') {
                        confValues = "./qa/values.yaml"
                        appName = app-qa
                        nameSpace = qa
                        dockertag = params.tagFromJob1
                        deploy(confValues, appName, nameSpace)
                    }
                }
                if (ischangeSetList()) {
                    stage('Deploy PROD release') {
                        def list = changeSetList()
                        def yamlFile = list[-1]
                        def dir = yamlFile.tokenize('/')
                        def nameSpace = dir[0]
                        def appNameWithExtention = dir[1]
                        def appName = appNameWithExtention.removeExtension()
                        deploy(confValues, appName, nameSpace)
                    }
                }
            }
        }
                def deploy(confValues, appName, nameSpace, dockerTag ) {
                    container('helm') {
                        withKubeConfig([credentialsId: 'kubeconfig']) {
                            sh """
                               helm upgrade --install $appName --namespace=$nameSpace --debug --force ./App/app --values $confValues \
                               --set image.tag="$dockertag"
                        """
                        }
                    }
                }


                boolean isMaster() {
                    return ("${params.tagFromJob1}" ==~  "/[a-z0-9]{7}/" )
                }
                boolean isBuildingTag() {
                    return ("${params.tagFromJob1}" ==~ /^v\d+.\d+.\d+$/ || "${params.tagFromJob1}" ==~ /^\d+.\d+.\d+$/ )
                }

                def ischangeSetList () {
                    def list
                    currentBuild.changeSets.each { changeSet ->
                        changeSet.items.each { entry ->
                            entry.affectedFiles.each { file ->
                                if (file.path ==~ /^prod-(ap1|eu1|us1|us2)\/\w+.yaml$/) {
                                    list.add(file.path)
                                }
                            }
                        }
                    }
                    return list
                }



