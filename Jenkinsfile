#!groovy

//update3

properties([buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '9', numToKeepStr: '8')), disableConcurrentBuilds(), pipelineTriggers([pollSCM('*/1 * * * *')])])

def checkout_git_code(String repo_url, String branch, String credentials_id,
  String target_directory, Boolean polling_enabled, String master) {
    checkout poll: polling_enabled, scm: [$class: 'GitSCM', branches: [[name: "*/${branch}"]],
      doGenerateSubmoduleConfigurations: false,
      extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: "${target_directory}"]],
      submoduleCfg: [],
      userRemoteConfigs: [[credentialsId: "${credentials_id}", url: "${repo_url}"]]]
 
  // Write Commit Id into the file.
  sh "cd ${target_directory} && git rev-parse --short HEAD > commit-id"
}



def create_docker_image() {
  sh """#!/bin/bash
  cd ${env.WORKSPACE}/sample-web-app/docker/
  
  echo "========Creating docker image========"
  docker build -t sanketgaykar/nginx:${env.IMAGE_TAG} .
  """
}
 
def push_docker_image() {
  sh """#!/bin/bash
  docker push sanketgaykar/nginx:${env.IMAGE_TAG}
  """
}


def create_helm_chart( ) {

  if ( "${env.BRANCH_NAME}" == "master"){
    env.chart_version = "0.1.0"
  }
  else {
    env.chart_version = "${env.BRANCH_NAME}".split(/-/)[1]

    if (!("${env.chart_version}" ==~ /[0-9]+(\.[0-9]+)(\.[0-9]+)?/ )){
      print "No symantic version found in branch name. Defaulting to 0.1.1"
      env.chart_version = "0.1.1"
    }
  }
  
  sh """#!/bin/bash

  cd ${env.WORKSPACE}/sample-web-app/helm-chart/web-app
 
  helm package ./ --app-version ${env.IMAGE_TAG} --version ${env.chart_version}

  #gsutil cp gs://backup-invisibly/devops/release/clearview/index.yaml .
  #helm repo index --url gs://backup-invisibly/devops/release/clearview/tag-manager-api/${env.chart_version}/ --merge index.yaml .

  #gsutil cp index.yaml gs://backup-invisibly/devops/release/clearview/
  #gsutil cp tag-manager-api-chart-${env.chart_version}.tgz gs://backup-invisibly/devops/release/clearview/tag-manager-api/${env.chart_version}/
  """
 
}

def branch_type="master"

env.BRANCH_NAME="master"

node() {
  try {
    stage('Checkout Source Code for deployment branch') {
      checkout_git_code('https://github.com/GaykarSanket/sample-app.git',
        "${env.BRANCH_NAME}", '', 'sample-web-app', false, 'master')
    }

    env.COMMIT_ID = readFile('sample-web-app/commit-id').trim()
    env.IMAGE_TAG="${env.BRANCH_NAME}-${env.COMMIT_ID}"

    stage('Create docker image') {
      create_docker_image()
    }
     
    stage("Push Docker Image to registry") {
      push_docker_image()
    }

    stage('Create helm chart'){
      create_helm_chart()
    }
	
	currentBuild.result = "SUCCESS"
  }
  catch (e) {
    currentBuild.result = "FAILED"
    throw e
  }
  finally {
    //Clean Workspace at end
    cleanWs()

    if (currentBuild.result == 'SUCCESS'){
    sh """#!/bin/bash
	cd ${env.WORKSPACE}/sample-web-app
	
	ls
	
	"""

	 build(job: "web-app-cd",
       parameters:
       [string(name: 'ENV', value: "dev"),
       string(name: 'IMAGE_TAG', value: "${env.IMAGE_TAG}")])
    }
  }
}
