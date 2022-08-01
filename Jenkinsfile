pipeline {
  agent {
    node {
      label 'nodejs'
    }
  }
  parameters{
     string(name:'TAG_NAME',defaultValue: '',description:'')
  }
  environment {
    DOCKERHUB_CREDENTIAL_ID = 'harbor-id'
    GITHUB_CREDENTIAL_ID = 'github-id'
    KUBECONFIG_CREDENTIAL_ID = 'kube-id'
    PRIVATE_REPO = '101.32.244.133'
    PROJECT = 'nasimobi'
    GTIHUB_ACCOUNT = 'howge'
    APP_NAME = 'devops-docs-sample'
  }
  stages {
    stage('checkout scm') {
      steps {
        checkout(scm)
      }
    }
    stage('get dependencies') {
      steps {
        container('nodejs') {
          sh 'npm install'
        }

      }
    }
    stage('unit test') {
      steps {
        container('nodejs') {
          sh 'yarn test'
        }

      }
    }
    stage('build & push snapshot') {
      steps {
        container('nodejs') {
          sh 'npm build'
          sh 'docker build -t $PRIVATE_REPO/$PROJECT/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER .'
          withCredentials([usernamePassword(passwordVariable : 'DOCKER_PASSWORD' ,usernameVariable : 'DOCKER_USERNAME' ,credentialsId : "$DOCKERHUB_CREDENTIAL_ID" ,)]) {
            sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'
            sh 'docker push  $PRIVATE_REPO/$PROJECT/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER '
          }
        }

      }
    }
    stage('push latest'){
       when{
         branch 'master'
       }
       steps{
         container('nodejs'){
           sh 'docker tag  $PRIVATE_REPO/$PROJECT/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER $PRIVATE_REPO/$PROJECT/$APP_NAME:latest '
           sh 'docker push  $PRIVATE_REPO/$PROJECT/$APP_NAME:latest '
         }
       }
    }
    stage('deploy to dev') {
      when{
        branch 'master'
      }
      steps {
        input(id: 'deploy-to-dev', message: 'deploy to dev?')
        kubernetesDeploy(configs: 'deploy/dev/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
      }
    }
    stage('push with tag'){
      when{
        expression{
          return params.TAG_NAME =~ /v.*/
        }
      }
      steps {
         container('nodejs'){
         input(id: 'release-image-with-tag', message: 'release image with tag?')
           withCredentials([usernamePassword(credentialsId: "$GITHUB_CREDENTIAL_ID", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
             sh 'git config --global user.email "anthony@nasimobi.com" '
             sh 'git config --global user.name "anthony" '
             sh 'git tag -a $TAG_NAME -m "$TAG_NAME" '
             sh 'git push https://$GIT_USERNAME:$GIT_PASSWORD@github.com/$GTIHUB_ACCOUNT/$APP_NAME.git --tags'
           }
         sh 'docker tag  $PRIVATE_REPO/$PROJECT/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER $PRIVATE_REPO/$PROJECT/$APP_NAME:$TAG_NAME '
         sh 'docker push  $PRIVATE_REPO/$PROJECT/$APP_NAME:$TAG_NAME '
         }
      }
    }
    stage('deploy to production') {
      when{
        expression{
          return params.TAG_NAME =~ /v.*/
        }
      }
      steps {
        input(id: 'deploy-to-production', message: 'deploy to production?')
        kubernetesDeploy(configs: 'deploy/prod/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
      }
    }
  }
}
