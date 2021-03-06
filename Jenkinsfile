#!groovy

import groovy.text.SimpleTemplateEngine
import org.jenkinsci.plugins.pipeline.modeldefinition.Utils

def label = "jenkins-agent2"

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
    jenkins: jenkins-agent2
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
""")

{ //pod template
node(label) {
def map = [
        'dev'     : [],
        'qa'      : [],
        'prod-ap1': [],
        'prod-eu1': [],
        'prod-us1': [],
        'prod-us2': []
]

    stage('Clone config repo') {
    checkout scm
    echo "tag from Job1 : ${params.tagFromJob1}"
}
def list = ischangeSetList()
println list
if (isMaster()) {
    map['dev'] = ['dev/hollychain.yaml']
}
if (isBuildingTag()) {
    map['qa'] =['qa/emptyworld.yaml']
}
if (list) {
    list.each { item ->
        def nameSpace = item.split('/')[0]
        map[nameSpace] = [item]
    }
}
    map.each {
        stage("Deploy release for " + it.key) {
            if (it.value) {
                deployStage(it.value)
            }
            else Utils.markStageSkippedForConditional("Deploy release for " + it.key)
        }
    }
} //node
} //pod template
def deployStage(list) {
    list.each { file_path ->
        def tag = params.tagFromJob1
        if (ischangeSetList()) {
            values = readYaml file: file_path
            tag = values.image.tag
        }
        def nameSpace = file_path.split('/')[0]
        def appName = file_path.split('/')[1].split(/\./)[0]
        checkoutConfRepo(tag)
        deploy(nameSpace, appName, file_path, tag)
    }
}
def checkoutConfRepo(tag) {

    checkout([$class           : 'GitSCM',
              branches         : [[name: tag]],
              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: tag]],
              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
}
def deploy( nameSpace, appName, file_path, tag ) {
    container('helm') {
        withKubeConfig([credentialsId: 'kubeconfig']) {
            sh """
               helm upgrade --install $appName --namespace=$nameSpace --debug --force ./$tag/app --values ./$file_path  \
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
                if (file.path ==~ /^prod-(ap1|eu1|us1|us2)\/\w+.yaml$/) {
                    list.add(file.path)
                }
            }
        }
    }
    return list.toSet()
}