
def last_stage

def is_release_branch
def is_master_branch
def run_log_file

def pomVersion

pipeline {
    agent any
    tools {
        maven '3.8.6'
    }
    stages {
        stage("enviroment settings"){
          steps{ 
             script {
                sh "printenv"
                run_log_file="/tmp/mscovid-${BUILD_TAG}.log"
                last_stage = env.STAGE_NAME
                is_release_branch = "${env.BRANCH_NAME}" ==~/release\/.*/
                is_master_branch = "${env.BRANCH_NAME}" == "master"  
                pomVersion = sh returnStdout: true, script: './mvnw  org.apache.maven.plugins:maven-help-plugin:evaluate -Dexpression=project.version |grep -v "[INFO]"'
                echo "version is ${pomVersion}"
             }
          }
        }
        stage("build & test") {
          when{ expression{ !is_master_branch } }
          steps{
            echo "build & test"
            script {
              last_stage = env.STAGE_NAME
              sh "./mvnw clean package -e"
            }
          }
        }

        stage('sonar') {
            when{ expression{ !is_master_branch } }
            steps {
                echo 'sonar...'
                script{
                last_stage = env.STAGE_NAME
                def scannerHome = tool 'local-sonar';
                withSonarQubeEnv(credentialsId:'sonartoken',installationName:'local-sonar') {
                   sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=lab-04 -Dsonar.java.binaries=build"
                }
                }
            }
        }
        

        stage("run"){
           when{ expression{ !is_master_branch } }
           steps{
             echo "run"
             script{
               last_stage = env.STAGE_NAME
               sh "./mvnw spring-boot:run  > ${run_log_file} 2>&1 &"
             }
           }
        }

        stage('wait serivice start') {
           when{ expression{ !is_master_branch  } }
           steps{
           timeout(5) {
             waitUntil {
               script {
                 def exitCode = sh script:"grep Started ${run_log_file}", returnStatus:true
                 return (exitCode == 0);
               }
             }
          }
          }
        }
        stage('test api rest') {
           when{ expression{ !is_master_branch } }
           steps{
               script { last_stage = env.STAGE_NAME  }
               echo 'test...'
               sh "curl -X GET 'http://localhost:8081/rest/mscovid/test?msg=testing'"
          }
        } 

        stage('update version and tag') {
           when{ expression{ is_master_branch } }
           steps{
              sh './mvnw -B -Darguments="-Dmaven.test.skip=true" -Dresume=false release:prepare release:perform'
           }        
        }
        stage('nexus') {
           when{ expression{ is_master_branch } }
           steps{
            script{ last_stage = env.STAGE_NAME }
            echo 'nexus...'
            step(
             [$class: 'NexusPublisherBuildStep',
                 nexusInstanceId: 'nexus01',
                 nexusRepositoryId: 'devops-usach-nexus',
                 packages: [[$class: 'MavenPackage',
                       mavenCoordinate: [artifactId: 'DevOpsUsach2020', groupId: 'com.devopsusach2020', packaging: 'jar', version: '${pomVersion}'],
                       mavenAssetList: [
                          [classifier: '', extension: 'jar', filePath: "${WORKSPACE}/build/DevOpsUsach2020-${pomVersion}.jar"]
                       ] 
                   ]
                 ]
               ]
             )
           }
        }
        stage('merge to main'){
           when{ expression{ is_release_branch } }
           steps{
              
              script { last_stage = env.STAGE_NAME  }
              git credentialsId: 'ssh_key', url: ' git@github.com:amillalen/ms-iclab.git', branch: 'master'
              sshagent(['ssh_key']) {
                sh 'git pull --all --no-rebase'
                sh 'git fetch --all'
                sh "git merge ${env.BRANCH_NAME}"
              }
              echo '${env.BRANCH_NAME}'
           }
        } 
        stage('Paso Notificación Slack') {
            steps {
                echo 'Notificando por Slack...'
                slackSend channel: 'D0435L5H7KJ', message: "[Grupo1][Pipeline IC/CD][Rama: ${env.BRANCH_NAME}][Stage: ${last_stage}][Resultado: Éxito/Success]."
            }
        }
      
    }   
    
    post {
        failure {
                slackSend channel: 'D0435L5H7KJ', message: "[Grupo1][Pipeline IC/CD][Rama: ${env.BRANCH_NAME}][Stage: ${last_stage}][Resultado: Error/Fail]."
        }
    }


} 
