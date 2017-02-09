#!/usr/bin/env groovy

// Parameters expected:
// githubCommit - sha of GitHub commit that is being tested

def project=""
def projectCreated=false
def appName="openshift-docs"
def projectPrefix="docs"

def setBuildStatus = { String context, String message, String state, String backref ->
     step([$class: "GitHubCommitStatusSetter",
           commitShaSource: [$class: "ManuallyEnteredShaSource", sha: "${githubCommit}"],
           contextSource: [$class: "ManuallyEnteredCommitContextSource", context: context],
           statusBackrefSource: [$class: "ManuallyEnteredBackrefSource", backref: ""],
           statusResultSource: [$class: "ConditionalStatusResultSource",
                                results: [[$class: "AnyBuildResult",
                                           message: message,
                                           state: state ]]]])
}

openshift.withCluster() {
    try {
        node {
            stage("Checkout") {
                setBuildStatus("ci/preview", "Generating preview", "PENDING", "${BUILD_URL}")
                checkout scm
            }
            stage ("Create Project") {
                project="docs-${githubCommit.substring(0,8)}"
                openshift.newProject(project.toString())
                projectCreated=true
            }
            openshift.withProject(project.toString()) {
                stage("Set Project Permissions") {
                    openshift.raw('policy', 'add-role-to-group',  'view', 'system:authenticated')
                }
                
                stage ("Apply object configurations") {
                    objs = openshift.process(readFile("_openshift/docs-template.yaml"))
                    openshift.apply(objs)
                }
                
                stage ("Build") {
                    openshift.startBuild(appName, "--from-repo=.", "--follow")
                }
                
                stage ("Verify Service") {
                    ep = openshift.selector("endpoints", appName)
                    timeout(5) {
                        ep.untilEach() {
                            return it.object().subsets.size() > 0
                        }
                    }
                }
                stage ("Preview Available") {
                    timeout(time:2, unit:'DAYS') {
                        input "Is everything ok?"
                    }
                }
            }
        }
    }
    finally {
        if (projectCreated) {
            node {
                stage('Delete PR Project') {
                    openshift.delete('project', project.toString())
                }
            }
        }
    }
}
