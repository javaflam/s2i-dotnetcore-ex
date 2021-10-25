def namespaceDev = "tnb-dev"
def namespaceTest = "tnb-test"
def namespaceProd = "tnb-prod"

def devTag = "dev-YYYY-MM-DD-HH-MM-SS"
def testTag = "test-YYYY-MM-DD-HH-MM-SS"
def prodTag = "prod-YYYY-MM-DD-HH-MM-SS"

pipeline {
  agent {
    kubernetes {
      cloud 'openshift'
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: jnlp
    image: image-registry.openshift-image-registry.svc:5000/openshift/jenkins-agent-base:latest
    command:
    args: ['\${computer.jnlpmac}', '\${computer.name}']
    workingDir: /tmp
    tty: false
  - name: skopeo
    image: quay.io/skopeo/stable:v1.2.0
    command:
    - cat
    tty: true
    resources:
      requests:
        memory: 128Mi
        cpu: 100m
  - name: dotnet
    image: image-registry.openshift-image-registry.svc:5000/openshift/dotnet:5.0
    command:
    - cat
    tty: true
    resources:
      requests:
        memory: 1Gi
        cpu: 100m
"""
    }
  }

  stages {
      stage('Checkout source code') {
        steps {
            git url: "${SOURCE_GIT_URL}", branch: "${SOURCE_GIT_BRANCH}"
            script{
                def buildTimeStamp = sh(script: "date +'%Y-%m-%d-%H-%M-%S'", returnStdout: true).trim()
                devTag = "dev-"+buildTimeStamp
                testTag = "test-"+buildTimeStamp
                prodTag = "prod-"+buildTimeStamp
            }
        }
      }
      stage('Restore') {
          steps {
              container("dotnet") {
                  echo "Restore app"
                  sh "dotnet restore app"
              }
          }
      }
      stage('Publish') {
          steps {
              container("dotnet") {
                  echo "Publish app"
                  sh "dotnet publish app --no-restore -c Release"
              }
          }
      }
      stage('Build and tag container image') {
          steps {
              echo "Building container image ${APP_NAME}:${devTag}"
              dir("${SOURCE_GIT_CONTEXT_DIR}") {
                  script {
                      openshift.withCluster() {
                          openshift.withProject("${namespaceDev}") {
                              def devBuildConfig = openshift.selector("bc", "${APP_NAME}").startBuild("--from-file=bin/Release/net5.0/publish", "--wait=true")
                              devBuildConfig.logs("-f")
                              echo "Tagging image ${APP_NAME}:${devTag}"
                              openshift.tag("${APP_NAME}:latest", "${APP_NAME}:${devTag}")
                          }
                      }
                  }
              }
          }
      }
      stage('Deploy to development project (Deployment)') {
          steps {
              echo "Deploying container image to development project ${namespaceDev}"
              script {
                  openshift.withCluster() {
                      openshift.withProject("${namespaceDev}") {
                          def devDeployment = openshift.selector("dc", "${APP_NAME}").object()
                          devDeployment.spec.template.spec.containers[0].image="image-registry.openshift-image-registry.svc:5000/${namespaceDev}/${APP_NAME}:${devTag}"
                          openshift.apply(devDeployment)
                          openshift.raw("rollout latest dc/${APP_NAME} -n ${namespaceDev}")
                          openshift.raw("rollout status dc/${APP_NAME} -n ${namespaceDev}")
                      }
                  }
              }
          }
      }
      stage('Tag container image for production') {
          steps {
              script {
                  openshift.withCluster() {
                      openshift.withProject("${namespaceDev}") {
                          openshift.tag("${APP_NAME}:${devTag}", "${APP_NAME}:${prodTag}")
                      }
                  }
              }
          }
      }
      stage('Publish production image to production project') {
          steps {
              echo "Publish production image to production project"
              script {
                  def openshift_token = sh([ script: 'oc whoami -t', returnStdout: true ]).trim()
                  container("skopeo") {
                      sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:${openshift_token} --dest-creds openshift:${openshift_token} docker://image-registry.openshift-image-registry.svc.cluster.local:5000/${namespaceDev}/${APP_NAME}:${prodTag} docker://image-registry.openshift-image-registry.svc.cluster.local:5000/${namespaceProd}/${APP_NAME}:${prodTag}"
                  }
              }
          }
      }
      stage('Deploy to production project (Deployment)') {
          steps {
              echo "Deploying container image to production project ${namespaceProd}"
              script {
                  openshift.withCluster() {
                      openshift.withProject("${namespaceProd}") {
                          def prodDeployment = openshift.selector("dc", "${APP_NAME}").object()
                          prodDeployment.spec.template.spec.containers[0].image="image-registry.openshift-image-registry.svc.cluster.local:5000/${namespaceProd}/${APP_NAME}:${prodTag}"
                          openshift.apply(prodDeployment)
                          openshift.raw("rollout latest dc/${APP_NAME} -n ${namespaceProd}")
                          openshift.raw("rollout status dc/${APP_NAME} -n ${namespaceProd}")
                      }
                  }
              }
          }
      }
  }
}
