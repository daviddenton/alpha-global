#!/usr/local/bin/groovy

def label = "worker-${UUID.randomUUID().toString()}"

def version = "latest"
def region = "eu-west-2"

def buildAndPush(String imageName, String version, String region) {
    stage("Build/push ${imageName} image") {
        container('docker') {
            withCredentials([string(credentialsId: 'aws_account_number', variable: 'awsAccountNumber')]) {
                def imageTag = "${awsAccountNumber}.dkr.ecr.${region}.amazonaws.com/${imageName}:${version}"
                sh "docker build -t ${imageTag} ${imageName}/."

                withAWS(credentials: 'aws_credentials') {
                    sh ecrLogin()
                    sh "docker push ${imageTag}"
                }
            }
        }
    }
}

def publishHelm() {
    stage('Build/push helm chart') {
        container('helm') {
            withCredentials([string(credentialsId: 'github_token', variable: 'githubToken')]) {
                sh "apk update"
                sh "apk add git"
                sh "git clone https://alphauser:${githubToken}@github.com/daviddenton/alpha-charts"
                sh "helm init"
                sh "helm package alpha-global-helm"
                sh "mv *.tgz alpha-charts"
                sh "helm repo index alpha-charts"
                sh "cd alpha-charts && git add . && git commit -am 'Auto Chart install' && git push"
            }
        }
    }
}

podTemplate(label: label, containers: [
        containerTemplate(name: 'gradle', image: 'gradle:4.5.1-jdk9', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.8.8', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:latest', command: 'cat', ttyEnabled: true)
],
        volumes: [
                hostPathVolume(mountPath: '/home/gradle/.gradle', hostPath: '/tmp/jenkins/.gradle'),
                hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
        ]) {
    node(label) {
        checkout([$class    : 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false,
                  extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/daviddenton/alpha-global']]])

        withCredentials([string(credentialsId: 'aws_account_number', variable: 'awsAccountNumber')]) {
            buildAndPush('alpha-global-app', version, region)
            buildAndPush('alpha-global-config', version, region)
            publishHelm()
        }
    }
}
