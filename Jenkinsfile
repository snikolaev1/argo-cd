import java.text.SimpleDateFormat

def l1 = 'dev'
def l2 = 'devx'
def serviceName = 'argocd'
def region = 'us-west-2'
def deployable_branches = ["master"]
def appName = ""
def kson_compnt= "sample"
def ptNameVersion = "${serviceName}-${UUID.randomUUID().toString().toLowerCase()}"
def repo = "dev/patterns/shivang-test-service-20/service"
def deploy_repo = "github.intuit.com/dev-devx/cdp-deployments.git"
def sample_app_deploy_repo = "github.intuit.com/sbseg-cdp/k8s-sample-app-deployment.git"
def tag = ""
def registry = "docker.artifactory.a.intuit.com"
def image = "${repo}/${serviceName}"
def app_wait_timeout = 600
def prd_diff_msg = ""
def stage_timeout = 20
def git_timeout = 2
def preprodOnly = true

// Admin
def argocd_server_admin = "aac963bf86dd111e887a30a00e274fcb-1107323148.us-west-2.elb.amazonaws.com:443"
def argocd_password_admin = "argocd-admin"
// Preprod
def iksType_preprod = 'preprod'
def argocd_server_preprod = "a3e82f0766f2911e8852f0677f8541ce-1604782097.us-west-2.elb.amazonaws.com:443"
def argocd_password_preprod = "argocd-preprod"

podTemplate(name: ptNameVersion, label: ptNameVersion, containers: [
    containerTemplate(name: 'cibuilder', image: 'argoproj/argo-cd-ci-builder:latest', ttyEnabled: true, command: 'cat', args: ''),
    containerTemplate(name: 'docker', image: 'docker:17-dind', ttyEnabled: true, alwaysPullImage: true, privileged: true, command: 'dockerd --host=unix:///var/run/docker.sock --host=tcp://0.0.0.0:2375 --storage-driver=overlay'),
    //containerTemplate(name: 'maven', image: 'maven:3.5-jdk-8', ttyEnabled: true, command: 'cat', args: ''),
    //containerTemplate(name: 'docker', image: 'docker:17.09', ttyEnabled: true, command: 'cat', args: '' ),
    containerTemplate(name: 'argocd', image: 'argoproj/argocd-cli:v0.4.7', ttyEnabled: true, command: 'cat', args: '' ),
    containerTemplate(name: 'cdtools', image: 'argoproj/argo-cd-tools:0.1.12', ttyEnabled: true, command: 'cat', args: ''),
    ],
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')],
    volumes: [emptyDirVolume(memory: false, mountPath: '/var/lib/docker')]
  )

{
    // DO NOT CHANGE
    def isPR = env.CHANGE_ID != null
    def branch = env.CHANGE_ID != null ? env.CHANGE_TARGET : env.BRANCH_NAME
    def dateFormat = new SimpleDateFormat("yyyyMMddHHmm")
    def date = new Date()
    def date_tag = dateFormat.format(date)

    // exit gracefully if not the master branch (or, rather, not in deployable_branches)
    if (!deployable_branches.contains(branch)) {
        stage("Skipping pipeline") {
            println "Branch: ${branch} is not part of deployable_branches"
            println "Skipping pipeline"
        }
        currentBuild.result = 'SUCCESS'
        return
    }

    node(ptNameVersion) {
        // DO NOT CHANGE
        def scmInfo = checkout scm
        def shortCommit = "${scmInfo.GIT_COMMIT}"[0..6]
        tag = "${env.BUILD_TAG}-${shortCommit}"
        def hasReleaseTag = sh(returnStdout: true, script: 'git tag --points-at HEAD').trim().startsWith('release-')

        // Build Stage
        stage('Build') {
                            container('cibuilder') {
                                sh ("docker version; mkdir -p /go/src/github.com/argoproj; ln -sf \$(pwd) /go/src/github.com/argoproj/argo-cd ; cd /go/src/github.com/argoproj/argo-cd; dep ensure && make controller-image server-image repo-server-image")
                            }
        }
        // Handle the PR build
        if (isPR) {
            stage("Skipping Deploy") {
                println "PR Builds: Skipping Deploy"
            }
            currentBuild.result = 'SUCCESS'
            return
        } // isPR
        else  {
            // The first milestone step starts tracking concurrent build order
            milestone()
            def env = "preprod"
            appName = "${l1}-${l2}-${serviceName}-${region}-${iksType_preprod}"
            // lock the Preprod Deploy and Test stages
            lock(resource: "${appName}-${env}", inversePrecedence: true) {
                timeout(time:"${stage_timeout}".toInteger(), unit:'MINUTES') {
                    // The Preprod Deploy stage
                    stage( "Deploy ${env}" ) {
                        withCredentials([string(credentialsId: "${argocd_password_admin}", variable: 'ARGOCD_PASS')]) {
                            container('argocd') {
                                println("Deploying to ${appName}")
                                //withCredentials([usernamePassword(credentialsId: 'github-svc-sbseg-ci', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                //    dir("deployment-${env}-${tag}") {
                                //        git url: "https://${deploy_repo}", credentialsId: "github-svc-sbseg-ci"
                                //        sh "/usr/local/bin/ks param set ${kson_compnt} image ${registry}/${image}:${tag} --env $env"
                                //        sh "git config --global user.email ${tag}@${ptNameVersion}"
                                //        sh "git config --global user.name ${tag}"
                                //        sh "git diff"
                                //        sh "git commit -am \"update container for ${env} during build ${tag}\""
                                //        lock("${deploy_repo}") {
                                //            timeout(time:"${git_timeout}".toInteger(), unit:'MINUTES') {
                                //                sh "git pull --rebase https://${GIT_USERNAME}:${GIT_PASSWORD}@${deploy_repo} master"
                                //                sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@${deploy_repo} master"
                                //            }
                                //        }
                                //    }
                                //}
                                sh "/argocd login ${argocd_server_admin} --name ${argocd_server_admin} --insecure  --username admin --password $ARGOCD_PASS"
                                //sh "/argocd install settings --superuser-password ${argocd_password_preprod} -n ${appName}-${env} --update-superuser"
                                sh "/argocd app create --name ${appName}-${env} --repo https://${deploy_repo} --path argocd --env ${env} --upsert"
                                sh "/argocd app sync ${appName}-${env}"
                                sh "/argocd app wait ${appName}-${env} --timeout ${app_wait_timeout}"
                            }
                            container('cdtools') {
                                sh "curl -k https://${argocd_server_preprod} 2>&1 | grep -q \"<title>Argo CD</title>\""
                            }
                        }
                    }
                    // The Preprod Test stage
                    appName = "${l1}-${l2}-sample-app-${region}-${iksType_preprod}"
                    stage( "test ${env}" ) {
                        withCredentials([string(credentialsId: "${argocd_password_preprod}", variable: 'ARGOCD_PASS')]) {
                            container('argocd') {
                                println("test to ${appName}")
                                //withCredentials([usernamePassword(credentialsId: 'github-svc-sbseg-ci', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                //    dir("deployment-${env}-${tag}") {
                                //        git url: "https://${deploy_repo}", credentialsId: "github-svc-sbseg-ci"
                                //        sh "/usr/local/bin/ks param set ${kson_compnt} image ${registry}/${image}:${tag} --env $env"
                                //        sh "git config --global user.email ${tag}@${ptNameVersion}"
                                //        sh "git config --global user.name ${tag}"
                                //        sh "git diff"
                                //        sh "git commit -am \"update container for ${env} during build ${tag}\""
                                //        lock("${deploy_repo}") {
                                //            timeout(time:"${git_timeout}".toInteger(), unit:'MINUTES') {
                                //                sh "git pull --rebase https://${GIT_USERNAME}:${GIT_PASSWORD}@${deploy_repo} master"
                                //                sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@${deploy_repo} master"
                                //            }
                                //        }
                                //    }
                                //}
                                sh "/argocd login ${argocd_server_preprod} --name ${argocd_server_preprod} --insecure  --username admin --password $ARGOCD_PASS"
                                withCredentials([string(credentialsId: "svc_argocd", variable: 'ARGOCD_TOKEN')]) {
                                    //sh "/argocd repo add https://${sample_app_deploy_repo} --username svc_argocd --password $ARGOCD_TOKEN"
                                    sh "/argocd repo add https://${sample_app_deploy_repo} --username svc_argocd --password $ARGOCD_TOKEN 2>&1 | grep -q http"
                                }
                                sh "/argocd app create --name ${appName}-${env} --repo https://${sample_app_deploy_repo} --path . --env qal --upsert"
                                sh "/argocd app sync ${appName}-${env}"
                                sh "/argocd app wait ${appName}-${env} --timeout ${app_wait_timeout}"
                            }
                            container('cdtools') {
                                sh "APP_NAME=${appName}-${env} ARGOCD_SERVER=\"https://${argocd_server_preprod}/api/v1\" USERNAME=admin PASSWORD=$ARGOCD_PASS HEALTH_URL_TEMPLATE=https://%s/health/full bash -x /usr/local/bin/ensure-service-up.sh"
                            }
                        }
                    }
                }
                milestone()
            }
        }
    }
    if (preprodOnly||isPR) {
        currentBuild.result = 'SUCCESS'
        return
    }
    stage('Deploy approval') {
        timeout(time:1, unit:'DAYS') {
            input "Deploy to Segment Instances?"
            submitter: "cblankenship,stsang,dthomson"
        }
        milestone()
    }
    node(ptNameVersion) {
        env = "prd"
        // lock the PRD Deploy and Test stages
        lock(resource: "${appName}-${env}", inversePrecedence: true) {
            timeout(time:"${stage_timeout}".toInteger(), unit:'MINUTES') {
                // The STG Deploy stage
                stage( "Deploy ${env}" ) {
                    withCredentials([string(credentialsId: "${argocd_password_admin}", variable: 'ARGOCD_PASS')]) {
                        container('argocd') {
                            println("Deploying to ${appName}")
                            withCredentials([usernamePassword(credentialsId: 'github-svc-sbseg-ci', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                dir("deployment-${env}-${tag}") {
                                    git url: "https://${deploy_repo}", credentialsId: "github-svc-sbseg-ci"
                                    sh "/usr/local/bin/ks param set ${kson_compnt} image ${registry}/${image}:${tag} --env $env"
                                    sh "git config --global user.email ${tag}@${ptNameVersion}"
                                    sh "git config --global user.name ${tag}"
                                    sh "git diff"
                                    sh "git commit -am \"update container for ${env} during build ${tag}\""
                                    lock("${deploy_repo}") {
                                        timeout(time:"${git_timeout}".toInteger(), unit:'MINUTES') {
                                            sh "git pull --rebase https://${GIT_USERNAME}:${GIT_PASSWORD}@${deploy_repo} master"
                                            sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@${deploy_repo} master"
                                        }
                                    }
                                }
                            }
                            sh ("/argocd login ${argocd_server_admin} --name context --insecure  --username admin --password $ARGOCD_PASS")
                            sh "/argocd app create --name ${appName}-${env} --repo https://${deploy_repo} --path . --env ${env} --upsert"
                            sh "/argocd app sync ${appName}-${env}"
                            sh "/argocd app wait ${appName}-${env} --timeout ${app_wait_timeout}"
                        }
                        container('cdtools') {
                            sh "APP_NAME=${appName}-${env} ARGOCD_SERVER=\"https://${argocd_server_admin}/api/v1\" USERNAME=admin PASSWORD=$ARGOCD_PASS HEALTH_URL_TEMPLATE=https://%s/health/full bash -x /usr/local/bin/ensure-service-up.sh"
                        }
                    }
                }
            }
            milestone()
        }
    }
}
