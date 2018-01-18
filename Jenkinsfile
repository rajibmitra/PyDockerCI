pipeline {
  options {
    buildDiscarder(logRotator(numToKeepStr:'100'))    
}
  agent none
  stages {
    stage('prep'){
      steps {
        node(label: 'dms') {
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
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/run_pytest_unit'
              junit 'report_pytest_unit.xml'
            }
          },
           // TODO: make the coverage report be built-into the unit/acceptance test steps
          'coverage': {
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/run_coverage'
              step([$class: 'CoberturaPublisher', autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: '**/report_coverage.xml', failUnhealthy: false, failUnstable: false, maxNumberOfBuilds: 0, onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false])
            }
          },
          'rest api html': {
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/scripts/build_rest_api_html'
              archiveArtifacts 'dms/scripts/docs/*.html'
              publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true, reportDir: 'dms/scripts/docs/', reportFiles: 'dms_rest_api.codegen_html.html', reportName: 'HTML Report', reportTitles: 'codegen_html'])
              publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true, reportDir: 'dms/scripts/docs/', reportFiles: 'dms_rest_api.pretty-swag.html', reportName: 'HTML Report', reportTitles: 'pretty-swag'])
            }
          },
          'tests acceptance': {
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/run_pytest_acceptance'
              junit allowEmptyResults: true, testResults: 'report_pytest_acceptance.xml'
            }
           },
          'flake8': {
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/run_flake8'
              warnings canComputeNew: false, canResolveRelativePaths: false, defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', parserConfigurations: [[parserName: 'Pep8', pattern: 'report_flake8.txt']], unHealthy: ''
            }
           },
          'pylint': {
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/run_pylint'
              warnings canComputeNew: false, canResolveRelativePaths: false, defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', parserConfigurations: [[parserName: 'PyLint', pattern: 'report_pylint.txt']], unHealthy: ''
            }
          },
          'clonedigger': {
             node(label: 'dms') {
               deleteDir()
               unstash 'code'
               sh 'dms/run_clonedigger'
               archiveArtifacts '*.html, *.xml'
               stash name: 'clonedigger_report', includes: 'report_clonedigger.xml'
             }
             node('master') {
             // This portion using a custom Warnings scanner for clonedigger parsing
             // can only run on the master with Warnings plugin 4.62.
             // See https://issues.jenkins-ci.org/browse/JENKINS-43813 for more info
               unstash 'clonedigger_report'
             warnings canComputeNew: false, canResolveRelativePaths: false, defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', parserConfigurations: [[parserName: 'CloneDigger', pattern: 'report_clonedigger.xml']], unHealthy: ''
            }
          },
          'build wheel package': {
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/build_wheel'
              archive 'dms/dist/*'
              stash name: 'wheel_package', includes: 'dms/dist/*.whl'
            }
          },
          'build sqlite3 database': {
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/build_sqlite3_db'
              archive 'dms/dms.sqlite3'
              stash name: 'sqlite3 database', includes: 'dms/dms.sqlite3'
            }
          },
          // UI stuff
          'ui build': {
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              sh 'dms/ui/build.sh'
              archiveArtifacts 'dms/ui/dist.tar.gz'
              stash name: 'covalent ui build tar package', includes: 'dms/ui/dist.tar.gz'
            }
          },
          'ui e2e tests': {
            node(label: 'dms') {
//              deleteDir()
//              unstash 'code'
//              // Run UI smoke test inline
//              wrap([$class: 'Xvfb']) {
//                lock('selenium_single_instance') {
//                  // lock so we do not accidentally run 2+ at a time on a single agent,
//                  // as we currently have a hard-coded TCP port
//                  sh 'dms/ui/smoke-test.sh'
//                }
//              }
//              junit 'dms/ui/junitresults.xml'
                echo 'disabled until we can get more stability'
            }
          }
        )
      }
    }
    stage('build docker images') {
      steps {
        parallel(
          'nginx UI docker image':{
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              unstash 'covalent ui build tar package'
              sh 'docker build -f dms_nginx_ui_Dockerfile -t dms-nginx-ui .'
              sh 'docker save dms-nginx-ui -o dms_nginx_ui_docker.tar'
              sh 'gzip dms_nginx_ui_docker.tar'
              archiveArtifacts 'dms_nginx_ui_docker.tar.gz'
              stash name : 'dms nginx UI docker image', includes: 'dms_nginx_ui_docker.tar.gz'
            }
          },
          'flask docker image':{
            node(label: 'dms') {
              deleteDir()
              unstash 'code'
              unstash 'wheel_package'
              unstash 'sqlite3 database'
              sh 'docker build -t flask_app -f dms_flask_Dockerfile .'
              sh 'docker save flask_app -o dms_flask_docker.tar'
              sh 'gzip dms_flask_docker.tar'
              archiveArtifacts 'dms_flask_docker.tar.gz'
              stash name: 'dms docker image', includes: 'dms_flask_docker.tar.gz'
            }
          }
        )
      }
    }
    stage('deploy to production'){
      when {
        // Skip deploy stage when not on 'master' Git branch
        branch 'master'
      }
      steps {
         node(label: 'dms') {
            deleteDir()
            unstash 'code'
            unstash 'wheel_package'
            unstash 'sqlite3 database'
            unstash 'covalent ui build tar package'
            sshagent(['4e2bbecf-6123-4dc9-8f6f-d3e5f7a1ac6a']) {
              lock('dms_production_system') {
                sh 'dms/run_dms_production_system_deployment --wheel_dist_dir dms/dist --sqlite_db_path dms/dms.sqlite3'
                dir ('ansible'){
                  sh 'ansible-playbook -i inventory dms_covalent_deployment.yml --extra-vars "dist_dir=../dms/ui/dist.tar.gz"'
                }
            }
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
