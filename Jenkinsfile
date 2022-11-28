
def last_stage

def is_release_branch
def is_master_branch
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
                last_stage = env.STAGE_NAME
                is_release_branch = "${env.BRANCH_NAME}" ==~/release\/.*/
                is_master_branch = "${env.BRANCH_NAME}" == "master"
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
               sh "./mvnw spring-boot:run  > /tmp/mscovid.log 2>&1 &"
             }
           }
        }

        stage('wait serivice start') {
           when{ expression{ !is_master_branch  } }
           steps{
           timeout(5) {
             waitUntil {
               script {
                 def exitCode = sh script:"grep -s Started /tmp/mscovid.log", returnStatus:true
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


        stage('nexus') {
           when{ expression{ !is_master_branch } }
           steps{
            script{ last_stage = env.STAGE_NAME }
            echo 'nexus...'
            step(
             [$class: 'NexusPublisherBuildStep',
                 nexusInstanceId: 'nexus01',
                 nexusRepositoryId: 'devops-usach-nexus',
                 packages: [[$class: 'MavenPackage',
                       mavenCoordinate: [artifactId: 'DevOpsUsach2020', groupId: 'com.devopsusach2020', packaging: 'jar', version: '0.0.1'],
                       mavenAssetList: [
                          [classifier: '', extension: 'jar', filePath: "${WORKSPACE}/build/DevOpsUsach2020-0.0.1.jar"]
                       ] 
                   ]
                 ]
               ]
             )
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
