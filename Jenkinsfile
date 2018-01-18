pipeline {

  agent none
  stages {
    stage('prep'){
      steps {
        node('master') {
          deleteDir()
          checkout scm
          stash 'code'
        }
      }
    }
    stage('build and testing') {
      steps {
        parallel(
          // REST API stuff
          'tests unit': {
            node('master') {
              deleteDir()
              unstash 'code'
              sh 'echo "HELLO"'
              
            }
          },
           // TODO: make the coverage report be built-into the unit/acceptance test steps
          'coverage': {
            node('master') {
              deleteDir()
              unstash 'code'
              sh 'echo "HELLO"'
            }
          },
          'tests acceptance': {
            node('master') {
              deleteDir()
              unstash 'code'
             sh 'echo "HELLO"'
            }
           },
          'flake8': {
            node('master') {
              deleteDir()
              unstash 'code'
              sh 'echo "HELLO"'
            }
           },
          'pylint': {
            node('master') {
              deleteDir()
              unstash 'code'
              sh 'echo "HELLO"'
            }
          },
         ,
          'build wheel package': {
            node('master') {
              deleteDir()
              unstash 'code'
              sh 'echo "HELLO"'
            }
          },
          'build sqlite3 database': {
            node('master') {
              deleteDir()
              unstash 'code'
              sh 'echo "HELLO"'
            }
          }
       
        )
      }
    }
    stage('build docker images') {
      steps {
        
          'flask docker image':{
            node('master') {
              deleteDir()
              sh 'echo "HELLO"'
            }
          }
        
      }
    }

  }
  post {
    failure {
      slackSend (color: '#FF0000', message: "${currentBuild.currentResult}:`${env.JOB_NAME}` #${env.BUILD_NUMBER}:\n${env.BUILD_URL}")
    }
    unstable {
      slackSend (color: '#FFFE89', message: "${currentBuild.currentResult}:`${env.JOB_NAME}` #${env.BUILD_NUMBER}:\n${env.BUILD_URL}")
    }
    changed{
        script {
            if(currentBuild.currentResult == 'SUCCESS') {
                if(env.BUILD_NUMBER != '1'){
                    slackSend (color: '#00FF00', message: "FIXED:`${env.JOB_NAME}` #${env.BUILD_NUMBER}:\n${env.BUILD_URL}")
                }
            }
        }
    }
   }
}
