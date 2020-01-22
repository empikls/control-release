#!groovy

import org.jenkinsci.plugins.pipeline.modeldefinition.Utils

def label = "jenkins"

properties([
        parameters([
                string(name: 'tagFromJob1', defaultValue: '', description: 'Short commit ID or Tag from upstream job', )
        ])
])


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
  - name: helm
    image: lachlanevenson/k8s-helm:v2.16.1
    command:
    - cat
    tty: true
"""
) {


                node(label) {
                    def tag =  params.tagFromJob1
                    def list = ischangeSetList()
                    def map = [
                            'dev'     : ['values': ''],
                            'qa'      : ['values': ''],
                            'prod-ap1': ['values': ''],
                            'prod-eu1': ['values': ''],
                            'prod-us1': ['values': ''],
                            'prod-us2': ['values': '']
                    ]
                    stage('Clone config repo') {
                        checkout scm
                        echo "tag from Job1 : ${params.tagFromJob1}"
                    }
                    if (isMaster()) {
                        map['dev'] = ['values': 'dev/values.yaml']
                    }
                    if (isBuildingTag()) {
                        map['qa'] = ['values': 'qa/values.yaml']
                    }
                    if (list) {
                        list.each { item ->
                            def nameSpace = item.split('/')[0]
                            def dockerTag = readYaml file: item
                            tag = dockerTag.image.tag
                            map[nameSpace] = ['values': item]
                        }
                    }
                    map.each {
                        stage("Deploy " + it.key) {
                            deployStage(it.value)
                        }
                    }
                }
            }
def deployStage(list) {
    list.each {
        def nameSpace = it.values.split('/')[0]
        def appName = it.values.split('/')[1].split(/\./)[0]
        checkoutConfRepo(tag)
        deploy(it.value, appName, nameSpace, tag)
    }
}
def checkoutConfRepo(tag) {
    checkout([$class           : 'GitSCM',
              branches         : [[name: branchName]],
              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: branchName]],
              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
    }
def deploy( it.value, appName, nameSpace, tag ) {
    container('helm') {
        withKubeConfig([credentialsId: 'kubeconfig']) {
            sh """
               helm upgrade --install $appName --namespace=$nameSpace --debug --force ./$tag/app --values $it.value  \
               --set image.tag=$tag
            """
        }
    }
}

boolean isMaster() {
    return ("${params.tagFromJob1}" ==~  /[a-z0-9]{7}/ )
}
boolean isBuildingTag() {
    return ("${params.tagFromJob1}" ==~ /^v\d+.\d+.\d+$/ || "${params.tagFromJob1}" ==~ /^\d+.\d+.\d+$/ )
}
def ischangeSetList() {
    def list = []
    currentBuild.changeSets.each { changeSet ->
        changeSet.items.each { entry ->
            entry.affectedFiles.each { file ->
                if (file.path ==~ /^prod-(ap1|eu1|us1|us2)\/\w+.yaml$/ ) {
                  list.add(file.path)
                }
            }
        }
    }
        return list.toSet()
}

