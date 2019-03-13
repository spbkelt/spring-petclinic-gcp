pipeline {
  agent any
  environment {
    ORG = 'spbkelt'
    PARENT_CHART_NAME = 'spring-petclinic-gcp'
    CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
  }
  stages {
    stage('CI Build and push snapshot') {
      when {
        branch 'PR-*'
      }
      environment {
        PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
        PREVIEW_NAMESPACE = "$PARENT_CHART_NAME-$BRANCH_NAME".toLowerCase()
        HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
      }
      steps {
        sh "mvn versions:set -DnewVersion=$PREVIEW_VERSION"
        sh "mvn install"
        sh "skaffold version"
        sh "export VERSION=$PREVIEW_VERSION && skaffold build -p dev -f skaffold.yaml"
        script {
          def services = ['spring-petclinic-api-gateway','spring-petclinic-customers-service','spring-petclinic-vets-service','spring-petclinic-visits-service']
          services.each { service ->
            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/${service}:$PREVIEW_VERSION"
          }
        }
        dir('charts/preview') {
          sh "make preview"
          sh "jx preview --app ${PARENT_CHART_NAME} --dir ../.."
        }
      }
    }
    stage('Build Release') {
      when {
        branch 'master'
      }
      steps {
        git 'https://github.com/spbkelt/spring-petclinic-gcp.git'

        // so we can retrieve the version in later steps
        sh "echo \$(jx-release-version) > VERSION"
        sh "mvn versions:set -DnewVersion=\$(cat VERSION)"
        sh "jx step tag --version \$(cat VERSION)"
        sh "mvn clean deploy"
        sh "skaffold version"
        sh "export VERSION=`cat VERSION` && skaffold build -p dev -f skaffold.yaml"
        script {
          def services = ['spring-petclinic-api-gateway','spring-petclinic-customers-service','spring-petclinic-vets-service','spring-petclinic-visits-service']
          services.each { service ->
            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/${service}:\$(cat VERSION)"
          }
        }
      }
    }
    stage('Promote to Environments') {
      when {
        branch 'master'
      }
      steps {
        dir('charts/spring-petclinic-gcp') {
          sh "jx step changelog --version v\$(cat ../../VERSION)"

          // release the helm chart
          sh "jx step helm release"

          // promote through all 'Auto' promotion Environments
          sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
        }
      }
    }
  }
}

