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
    string(name: 'DockerHubUsername', defaultValue: 'vecinomio', description: 'Enter your Dockerhub username')
    string(name: 'Email', defaultValue: 'vecinomio@gmail.com', description: 'Enter the desired Email for the Job notifications')
    string(name: 'Version', defaultValue: '1.0.0', description: 'Change version of the App')
    booleanParam(name: 'Build', defaultValue: true, description: 'Specify to Build App and do Tests')
    booleanParam(name: 'Release', defaultValue: false, description: 'Specify to deliver an artifact to ECR and tags to Github repos')
    booleanParam(name: 'Deployment', defaultValue: false, description: 'Specify to Deploy a new version of App to ECS')
    // choice(name: 'NewVersionTrafficWeight', choices: getWeight(), description: 'Set the amount of traffic for the new version in %. \nExample: If choose 10, than 10% of the traffic will forward to the new version, and 90% to the current one.')
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
            sh 'cd sa-frontend && npm run build'
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
    // stage("Tagging") {
    //   when { environment name: 'Release', value: 'true' }
    //   steps {
    //     script {
    //       try {
    //         sh "git tag -a ${Tag} -m 'Added tag ${Tag}'"
    //         sh "git push origin ${Tag}"
    //         sh "rm -rf ${OPSRepoBranch}"
    //         sh "mkdir -p ${OPSRepoBranch}"
    //         dir("${OPSRepoBranch}") {
    //           git(url: "${OPSRepoURL}", branch: "${OPSRepoBranch}", credentialsId: "devopsa3")
    //           sshagent (credentials: ['devopsa3']) {
    //             sh "git tag -a ${Tag} -m 'Added tag ${Tag}'"
    //             sh "git push origin ${Tag}"
    //           }
    //         }
    //         currentBuild.result = 'SUCCESS'
    //       }
    //       catch (err) {
    //         removeUnusedImages()
    //         sh "rm -rf ${OPSRepoBranch}"
    //         currentBuild.result = 'FAILURE'
    //         emailext body: "${err}. Tagging Stage Failed, check logs.", subject: "${FailureEmailSubject}", to: "${Email}"
    //         throw (err)
    //       }
    //       echo "result is: ${currentBuild.currentResult}"
    //     }
    //   }
    // }
    stage("CleanUp") {
      steps {
        removeUnusedImages()
      }
    }
    // stage("Deployment") {
    //   when { environment name: 'Deployment', value: 'true' }
    //   steps {
    //     script {
    //       try {
    //           UnicId = "${Tag}".replaceAll("\\.", "-")
    //           sh """
    //              CurrentStack=\$(aws cloudformation describe-stacks --output text --query "Stacks[?contains(StackName,'ECS-task')].[StackName]" --region ${AWSRegion} | tail -1)
    //              CurrentDeploymentColor=\$(aws cloudformation describe-stacks --stack-name \$CurrentStack --query "Stacks[].Parameters[?ParameterKey=='DeploymentColor'].ParameterValue" --output text --region ${AWSRegion} | tail -1)
    //              NewDeploymentColor="Green"
    //              if [ \$CurrentDeploymentColor == "Green" ]
    //                  then
    //                      NewDeploymentColor="Blue"
    //              fi
    //              aws cloudformation deploy --stack-name ECS-task-${UnicId} --template-file ops/cloudformation/ECS/ecs-task.yml --parameter-overrides ImageUrl=${ECRURI}/${AppRepoName}:${Tag} ServiceName=snakes-${UnicId} DeploymentColor=\$NewDeploymentColor --capabilities CAPABILITY_IAM --region ${AWSRegion}
    //              aws cloudformation deploy --stack-name alb --template-file ops/cloudformation/alb.yml --parameter-overrides VPCStackName=DevVPC \${CurrentDeploymentColor}Weight=${CurrentVersionTrafficWeight} \${NewDeploymentColor}Weight=${NewVersionTrafficWeight} --capabilities CAPABILITY_IAM --region ${AWSRegion}
    //              """
    //         currentBuild.result = 'SUCCESS'
    //         emailext body: 'New release was successfully deployed to ECS.', subject: "${SuccessEmailSubject}", to: "${Email}"
    //       }
    //       catch (err) {
    //         currentBuild.result = 'FAILURE'
    //         emailext body: "${err}. ECS Stack Creation Failed, check logs.", subject: "${FailureEmailSubject}", to: "${Email}"
    //         throw (err)
    //       }
    //       echo "result is: ${currentBuild.currentResult}"
    //     }
    //   }
    // }
  }
}
