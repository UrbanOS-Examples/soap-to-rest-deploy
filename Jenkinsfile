library(
    identifier: 'pipeline-lib@4.6.1',
    retriever: modernSCM([$class: 'GitSCMSource',
                          remote: 'https://github.com/SmartColumbusOS/pipeline-lib',
                          credentialsId: 'jenkins-github-user'])
)

properties([
    pipelineTriggers([scos.dailyBuildTrigger()]),
    parameters([
        booleanParam(defaultValue: false, description: 'Deploy to development environment?', name: 'DEV_DEPLOYMENT'),
        string(defaultValue: 'development', description: 'Image tag to deploy to dev environment', name: 'DEV_IMAGE_TAG')
    ])
])

def image
def doStageIf = scos.&doStageIf
def doStageIfDeployingToDev = doStageIf.curry(env.DEV_DEPLOYMENT == "true")
def doStageIfMergedToMaster = doStageIf.curry(scos.changeset.isMaster && env.DEV_DEPLOYMENT == "false")
def doStageIfRelease = doStageIf.curry(scos.changeset.isRelease)

node ('infrastructure') {
    ansiColor('xterm') {
        scos.doCheckoutStage()


        doStageIfDeployingToDev('Deploy to Dev') {
            deployTo(environment: 'dev', extraArgs: "--set image.tag=${env.DEV_IMAGE_TAG} --set image.pullPolicy=Always --recreate-pods")
        }

        doStageIfMergedToMaster('Deploy to Staging')  {
            deployTo(environment: 'staging')
            scos.applyAndPushGitHubTag('staging')
        }

        doStageIfRelease('Deploy to Production') {
            deployTo(environment: 'prod', internal: false)
            scos.applyAndPushGitHubTag('prod')
        }
    }
}

def deployTo(params = [:]) {
    def environment = params.get('environment')
    if (environment == null) throw new IllegalArgumentException("environment must be specified")

    def internal = params.get('internal', true)
    def extraArgs = params.get('extraArgs', '')

    scos.withEksCredentials(environment) {
        sh("""#!/bin/bash
            set -e
            helm init --client-only
            helm upgrade --install soap-to-rest ./chart \
                --namespace=vendor-resources \
                --values=soap-to-rest.yaml \
                ${extraArgs}
        """.trim())
    }
}
