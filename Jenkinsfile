#!/usr/bin/env groovy

// Pipeline variables
def isPR=false                   // true if the branch being tested belongs to a PR
def project=""                   // project where build and deploy will occur
def projectCreated=false         // true if a project was created by this build and needs to be cleaned up
def repoUrl=""                   // the URL of this project's repository
def appName="openshift-docs"     // name of application to create
def approved=false               // true if the preview was approved
def commitId=""

// getRepoURL retrieves the origin URL of the current source repository
def getRepoURL = {
  return sh(script: "git config --get remote.origin.url", returnStdout: true).trim()
}

def getRepoCommit = {
  return sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
}

// getRouteHostname retrieves the host name from the given route in an
// OpenShift namespace
def getRouteHostname = { String routeName, String projectName ->
  return sh(script: "oc get route ${routeName} -n ${projectName} -o jsonpath='{ .spec.host }'", returnStdout: true).trim()
}

try { // Use a try block to perform cleanup in a finally block when the build fails

  node {
    // Initialize variables in default node context
    isPR        = env.BRANCH_NAME ? env.BRANCH_NAME.startsWith("PR") : false
    baseProject = env.PROJECT_NAME
    project     = env.PROJECT_NAME

    stage ('Checkout') {
      checkout scm
      repoUrl = getRepoURL()
      commitId = getRepoCommit()
    }

    // When testing a PR, create a new project to perform the build
    // and deploy artifacts.
    stage ('Create PR Project') {
      project = uniqueName("${appName}-${commitId}")
      sh "oc new-project ${project}"
      projectCreated=true
      sh "oc policy add-role-to-group view system:authenticated -n ${project}"
    }

    stage ('Apply object configurations') {
      sh "oc process -f _openshift/docs-template.yaml -n ${project} | oc apply -f - -n ${project}"
    }

    stage ('Build') {
      sh "oc start-build ${appName} -n ${project} --from-repo=. --follow"
    }


    stage ('Verify Service') {
      openshiftVerifyService serviceName: appName, namespace: project
    }
    def appHostName = getRouteHostname(appName, project)
    stage ('Manual Test') {
      timeout(time:2, unit:'DAYS') {
        input "Is everything OK?"
      }
    }
    approved = true
  }
}
finally {
  if (projectCreated) {
    node {
      stage('Delete PR Project') {
        sh "oc delete project ${project}"
      }
    }
  }
}
