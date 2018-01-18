pipeline {
  options {
    buildDiscarder(logRotator(numToKeepStr:'10'))    
}
  agent none
  stages {
    stage('prep'){
      steps {
        node(label: 'master') {
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
            node(label: 'master') {
              deleteDir()
              unstash 'code'
              sh 'run_pytest_unit'
              #junit 'report_pytest_unit.xml'
            }
          },
           // TODO: make the coverage report be built-into the unit/acceptance test steps
          'coverage': {
            node(label: 'master') {
              deleteDir()
              unstash 'code'
              sh 'run_coverage'
              #step([$class: 'CoberturaPublisher', autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: '**/report_coverage.xml', failUnhealthy: false, failUnstable: false, maxNumberOfBuilds: 0, onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false])
            }
          },
          'tests acceptance': {
            node(label: 'master') {
              deleteDir()
              unstash 'code'
              sh 'run_pytest_acceptance'
             # junit allowEmptyResults: true, testResults: 'report_pytest_acceptance.xml'
            }
           },
          'flake8': {
            node(label: 'master') {
              deleteDir()
              unstash 'code'
              sh 'run_flake8'
              #warnings canComputeNew: false, canResolveRelativePaths: false, defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', parserConfigurations: [[parserName: 'Pep8', pattern: 'report_flake8.txt']], unHealthy: ''
            }
           },
          'pylint': {
            node(label: 'master') {
              deleteDir()
              unstash 'code'
              sh 'run_pylint'
              #warnings canComputeNew: false, canResolveRelativePaths: false, defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', parserConfigurations: [[parserName: 'PyLint', pattern: 'report_pylint.txt']], unHealthy: ''
            }
          },
         ,
          'build wheel package': {
            node(label: 'master') {
              deleteDir()
              unstash 'code'
              sh 'build_wheel'
              archive 'dist/*'
              #stash name: 'wheel_package', includes: 'dist/*.whl'
            }
          },
          'build sqlite3 database': {
            node(label: 'master') {
              deleteDir()
              unstash 'code'
              sh 'build_sqlite3_db'
              archive 'dms.sqlite3'
              #stash name: 'sqlite3 database', includes: 'dms.sqlite3'
            }
          },
       
        )
      }
    }
    stage('build docker images') {
      steps {
        
          'flask docker image':{
            node(label: 'master') {
              deleteDir()
              unstash 'code'
              unstash 'wheel_package'
              unstash 'sqlite3 database'
              sh 'docker build -t flask_app -f dms_flask_Dockerfile .'
              sh 'docker save flask_app -o dms_flask_docker.tar'
              sh 'gzip dms_flask_docker.tar'
             # archiveArtifacts 'dms_flask_docker.tar.gz'
              #stash name: 'dms docker image', includes: 'dms_flask_docker.tar.gz'
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
