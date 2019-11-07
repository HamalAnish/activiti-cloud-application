pipeline {
    agent {
      label "jenkins-maven-java11"
    }
    environment {
      ORG               = 'activiti'
      APP_NAME          = 'activiti-cloud-dependencies'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
      PREVIEW_NAMESPACE = "example-$BRANCH_NAME-$BUILD_NUMBER".toLowerCase()
      GLOBAL_GATEWAY_DOMAIN="35.228.195.195.nip.io"
      REALM = "activiti"
      GATEWAY_HOST = "gateway.$PREVIEW_NAMESPACE.$GLOBAL_GATEWAY_DOMAIN"
      SSO_HOST = "identity.$PREVIEW_NAMESPACE.$GLOBAL_GATEWAY_DOMAIN"
      HELM_RELEASE_NAME = "example-$BRANCH_NAME-$BUILD_NUMBER".toLowerCase()
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container('maven') {
            sh "git config --global credential.helper store" 
            sh "jx step git credentials"
            sh "mvn versions:set -DnewVersion=$PREVIEW_VERSION"
            sh "mvn install"
            sh "make updatebot/push-version-dry"
            sh "make run-full-chart"  
            dir("./activiti-cloud-acceptance-scenarios") {
               git 'https://github.com/Activiti/activiti-cloud-acceptance-scenarios.git'
               sh 'sleep 30'
               sh "mvn clean install -DskipTests && mvn -pl 'runtime-acceptance-tests,modeling-acceptance-tests' clean verify"
             }
          }
        }
        post {
                always {
                  delete_deployment()
                }
              }
      }
      stage('Build Release') {
        when {
          branch 'develop'
        }
        steps {
          container('maven') {
            // ensure we're not on a detached head
            sh "git checkout develop"
            sh "git config --global credential.helper store"

            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            sh "mvn versions:set -DnewVersion=\$(cat VERSION)"
            sh "mvn clean verify"

            retry(5){
              sh "git add --all"
              sh "git commit -m \"Release \$(cat VERSION)\" --allow-empty"
              sh "git tag -fa v\$(cat VERSION) -m \"Release version \$(cat VERSION)\""
              sh "git push origin v\$(cat VERSION)"
            }
          }
          container('maven') {
            sh 'mvn clean deploy'

            sh 'export VERSION=`cat VERSION`'
            sh 'export UPDATEBOT_MERGE=false'

            sh "jx step git credentials"

            retry(2){
                sh "make updatebot/push-version"
            }
            sh "make update-ea"  

          }
        }
      }
      stage('Build Release from Tag') {
        when {
          tag '*RELEASE'
        }
        steps {
          container('maven') {
            // ensure we're not on a detached head
            sh "git checkout $TAG_NAME"
            sh "git config --global credential.helper store"

            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo \$TAG_NAME > VERSION"
            sh "mvn versions:set -DnewVersion=\$(cat VERSION)"
          }
          container('maven') {
            sh '''
              mvn clean deploy -P !alfresco -P central
              '''

            sh 'export VERSION=`cat VERSION`'// && skaffold build -f skaffold.yaml'

            sh "git config --global credential.helper store"

            sh "jx step git credentials"
            //sh "updatebot push"
            //sh "updatebot update"

            sh "echo pushing with update using version \$(cat VERSION)"

            //add updatebot configuration to push to downstream
            sh "updatebot push-version --kind maven org.activiti.cloud.dependencies:activiti-cloud-dependencies \$(cat VERSION)"
          }
        }
      }
    }
    post {
        always {
            cleanWs()
        }
    }
  }
def delete_deployment() {
  container('maven') {
    dir("./charts/$APP_NAME") {
      sh "make delete"
    }
    sh "kubectl delete namespace $PREVIEW_NAMESPACE"
  }
}
