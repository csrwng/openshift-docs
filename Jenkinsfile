#!/usr/bin/env groovy

// Parameters expected:
// githubCommit - sha of GitHub commit that is being tested

def project=""
def projectCreated=false
def appName="openshift-docs"
def projectPrefix="docs"
def previewGenerated=false

def setBuildStatus = { String context, String message, String state, String backref ->
     step([$class: "GitHubCommitStatusSetter",
           commitShaSource: [$class: "ManuallyEnteredShaSource", sha: "${githubCommit}"],
           contextSource: [$class: "ManuallyEnteredCommitContextSource", context: context],
           statusBackrefSource: [$class: "ManuallyEnteredBackrefSource", backref: backref],
           statusResultSource: [$class: "ConditionalStatusResultSource",
                                results: [[$class: "AnyBuildResult",
                                           message: message,
                                           state: state ]]]])
}

openshift.withCluster() {
    try {
        node {
            stage("Checkout") {
                setBuildStatus("ci/preview", "Generating preview", "PENDING", "${RUN_DISPLAY_URL}")
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
                    def route = openshift.selector("route", appName)
                    def hostName = route.object().spec.host
                    setBuildStatus("ci/preview", "Preview available.", "PENDING", "${RUN_DISPLAY_URL}")
                    setBuildStatus("ci/preview/docs", "Click on Details to see preview.", "PENDING", "http://{hostName}/index.html")
                    previewGenerated=true
                    timeout(time:2, unit:'DAYS') {
                        input "Done with preview?"
                    }
                    
                }
            }
        }
    }
    finally {
        if (projectCreated) {
            node {
                stage('Cleanup') {
                    openshift.delete('project', project.toString())
                }
            }
        }
        if (previewGenerated) {
            setBuildStatus("ci/preview", "Done.", "SUCCESS", "")
            setBuildStatus("ci/preview/docs", "Preview no longer available.", "SUCCESS", "")
        } else {
            setBuildStatus("ci/preview", "Failed to generate preview.", "FAILURE", "")
        }
    }
}
