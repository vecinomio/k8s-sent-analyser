#!groovy
//Only one build can run
properties([disableConcurrentBuilds()])

def removeUnusedImages() {
  sh 'docker image prune -af --filter="label=maintainer=imaki"'
}

pipeline {
  agent {
    label 'master'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    timestamps()
  }
  parameters {
    string(name: 'DockerHubUsername', defaultValue: 'vecinomio', description: 'Enter your DockerHub username')
    string(name: 'Email', defaultValue: 'vecinomio@gmail.com', description: 'Enter the desired Email for the Job notifications')
    string(name: 'Version', defaultValue: '1.0.0', description: 'Change version of the App to desired')
    booleanParam(name: 'Build', defaultValue: true, description: 'Specify to Build App and do Tests')
    booleanParam(name: 'Release', defaultValue: false, description: 'Specify to deliver an image to the DockerHub')
    booleanParam(name: 'Deployment', defaultValue: false, description: 'Specify to Deploy a new version of App to GKE')
  }
  environment {
    DockerHubUsername = "${params.DockerHubUsername}"
    DockerHubCreds = "dockerHub"
    AppRepoName = 'sa-frontend'
    BuildAndTest = "${params.Build}"
    Release = "${params.Release}"
    Deployment = "${params.Deployment}"
    Tag = "${params.Version}"
    Email = "${params.Email}"
    FailureEmailSubject = "JOB with identifier ${Tag} FAILED"
    SuccessEmailSubject = "JOB with identifier ${Tag} SUCCESS"
  }

  stages {
    stage("Condition") {
      steps {
        script {
          if (Release == 'false') {
            Tag = "${BUILD_NUMBER}"
          }
        }
      }
    }
    stage("Build") {
      when { environment name: 'BuildAndTest', value: 'true' }
      steps {
        script {
          try {
            dir('sa-frontend') {
              sh 'npm run build'
            }
            currentBuild.result = 'SUCCESS'
          }
          catch (err) {
            currentBuild.result = 'FAILURE'
            emailext body: "${err}. Build Application Failed, check logs.", subject: "${FailureEmailSubject}", to: "${Email}"
            throw (err)
          }
          echo "result is: ${currentBuild.currentResult}"
        }
      }
    }
    stage("Build Docker Image") {
      when { environment name: 'BuildAndTest', value: 'true' }
      steps {
        script {
          try {
            dir('sa-frontend') {
              dockerImage = docker.build("${DockerHubUsername}/${AppRepoName}:${Tag}")
            }
            currentBuild.result = 'SUCCESS'
          }
          catch (err) {
            removeUnusedImages()
            currentBuild.result = 'FAILURE'
            emailext body: "${err}. Build Docker Image Failed, check logs.", subject: "${FailureEmailSubject}", to: "${Email}"
            throw (err)
          }
          echo "result is: ${currentBuild.currentResult}"
        }
      }
    }
    stage("Test") {
      when { environment name: 'BuildAndTest', value: 'true' }
      steps {
        script {
          try {
            testContainer = dockerImage.run('-p 8092:80 --name test')
            retry(10) {
              sh 'sleep 5'
              sh 'curl -sS http://localhost:8092 | grep "Sentiment Analysis"'
            }
            testContainer.stop()
            currentBuild.result = 'SUCCESS'
          }
          catch (err) {
            testContainer.stop()
            removeUnusedImages()
            currentBuild.result = 'FAILURE'
            emailext body: "${err}. Test Failed, check logs.", subject: "${FailureEmailSubject}", to: "${Email}"
            throw (err)
          }
        }
      }
    }
    stage("Delivery") {
      when { environment name: 'Release', value: 'true' }
      steps {
        script {
          try {
            docker.withRegistry("", DockerHubCreds ) {
              dockerImage.push()
            }
            currentBuild.result = 'SUCCESS'
          }
          catch (err) {
            removeUnusedImages()
            currentBuild.result = 'FAILURE'
            emailext body: "${err}. Delivery to ECR Failed, check logs.", subject: "${FailureEmailSubject}", to: "${Email}"
            throw (err)
          }
          echo "result is: ${currentBuild.currentResult}"
        }
      }
    }
    stage("CleanUp") {
      steps {
        removeUnusedImages()
      }
    }
    stage("Deployment") {
      when { environment name: 'Deployment', value: 'true' }
      steps {
        script {
          try {
              sh "kubectl set image deployment/${AppRepoName} ${AppRepoName}=${DockerHubUsername}/${AppRepoName}:${Tag}"
            currentBuild.result = 'SUCCESS'
            emailext body: 'New release was successfully deployed to GKE.', subject: "${SuccessEmailSubject}", to: "${Email}"
          }
          catch (err) {
            currentBuild.result = 'FAILURE'
            emailext body: "${err}. Deploy to GKE Failed, check logs.", subject: "${FailureEmailSubject}", to: "${Email}"
            throw (err)
          }
          echo "result is: ${currentBuild.currentResult}"
        }
      }
    }
  }
}
