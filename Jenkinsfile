node {
    def mvnHome
    mvnHome = tool 'mvn'
    env.PATH = "${mvnHome}/bin:${env.PATH}"
    
    def vmName
    vmName = "Debian-Runner-${BUILD_NUMBER}-TJ"
       
    def ip
    ip = "172.16.20.92"
    
    stage('Preparation') { 
        git url:'https://github.com/hfo/spring-petclinic.git'
    }
    
    stage('Build&Upload Maven Artifact(jar)') {
        sh './mvnw clean install -Dmaven.test.skip=true'

        sh "./mvnw sonar:sonar -Dsonar.host.url=http://172.16.20.157:9000"
        
        build job: 'nexus_uploader_job'
    }
    
    
    stage('Build&Upload Docker Image'){
        
        sh 'rm -f Dockerfile'
        sh 'rm -f petclinic-1.0.0.jar'
        
        sh'wget http://172.16.20.157:8081/repository/Jenkins-Repo/Dockerfile'
        sh'wget http://172.16.20.157:8081/repository/Jenkins-Repo/de/proficom/cdp/petclinic/1.0.0/petclinic-1.0.0.jar'
        
        sh 'docker build -t petclinic_alpine .'

       sh 'docker tag petclinic_alpine 172.16.20.157:8082/petclinic_alpine'
       sh 'docker push 172.16.20.157:8082/petclinic_alpine'
   }
    
   //checkpoint 'before Create VM & Deploy App'
    
   stage('Create VM & Deploy App'){
       //input id: 'Wait-for-manual-continue-0', message: 'Waiting for manual continue' 
       
       def hook
       hook = registerWebhook()
       
       build job: 'oo_create_runner_vm_from_template', parameters: [[$class: 'StringParameterValue', name: 'hook_url', value: hook.getURL()],[$class: 'StringParameterValue', name: 'vmName', value: vmName],[$class: 'StringParameterValue', name: 'IP_new', value: ip],[$class: 'StringParameterValue', name: 'pipelineBuildNumber', value: BUILD_NUMBER]]
       
       //sh 'docker pull 172.16.20.157:8082/petclinic_alpine' 
       //sh 'docker run -d -p 4100:8080 petclinic_alpine java -jar /usr/src/petclinic/petclinic-1.0.0.jar'
 
       echo "Waiting for POST to ${hook.getURL()}"
       
       def data
       data = waitForWebhook hook
       
       def str
       str = data.split('&')
       
       def messageStr
       messageStr = str[0].split('=')
       
       def statusStr
       statusStr = str[1].split('=')
       
       if( statusStr[1].equals("success")) { 
           echo "Webhook was called, VM was removed succesfully. Message: ${messageStr[1]}"
           
           build job: 'deployPetclinicDockerSSH', parameters: [[$class: 'StringParameterValue', name: 'ip', value: ip]]
       }else{
           echo "Webhook was called, VM was removed NOT succesfully. Message: ${messageStr[1]}"
           
           def hook3
           hook3 = registerWebhook()

           build job: 'oo_remove_runner_vm', parameters: [[$class: 'StringParameterValue', name: 'hook_url', value: hook3.getURL()],[$class: 'StringParameterValue', name: 'vmName', value: vmName],[$class: 'StringParameterValue', name: 'pipelineBuildNumber', value: BUILD_NUMBER]]

           echo "Waiting for POST to ${hook3.getURL()}"

           def data3
           data3 = waitForWebhook hook3

           def str3
           str3 = data3.split('&')

           def messageStr3
           messageStr3 = str3[0].split('=')

           def statusStr3
           statusStr3 = str3[1].split('=')

           if( statusStr3[1].equals("success")) { 
               echo "Webhook was called, VM was removed succesfully. Message: ${messageStr3[1]}"
               //quit the pipleine forcefully as the parent step failed
               sh "exit 1"
           }else{
               echo "Webhook was called, VM was removed NOT succesfully. Message: ${messageStr3[1]}. The VM might NOT have been removed from system!"
               //quit the pipleine forcefully as this step and the parent step failed
               sh "exit 1"
           }
       }
       
       //sh 'ssh administrator@172.16.20.93 "rm -f petclinic-1.0.0.jar; wget http://172.16.20.92:8081/repository/Jenkins-Repo/de/proficom/cdp/petclinic/1.0.0/petclinic-1.0.0.jar; ls"'
       //sh 'ssh administrator@172.16.20.93 "nohup java -jar petclinic-1.0.0.jar &"'
       
       //input id: 'Wait-for-manual-continue-1', message: 'Waiting for manual continue' 
   }
    
   stage('Functional tests'){
      //build job: 'UFT Test'
   }
    
   stage('Performance tests'){
       //build job: 'LR_Test_Job_2'
     
   }
    
   stage('Clean up testenvironment'){
       //input id: 'Wait-for-manual-continue-2', message: 'Waiting for manual continue' 
       
       def hook2
       hook2 = registerWebhook()
   
       build job: 'oo_remove_runner_vm', parameters: [[$class: 'StringParameterValue', name: 'hook_url', value: hook2.getURL()],[$class: 'StringParameterValue', name: 'vmName', value: vmName],[$class: 'StringParameterValue', name: 'pipelineBuildNumber', value: BUILD_NUMBER]]
       
       echo "Waiting for POST to ${hook2.getURL()}"
       
       def data2
       data2 = waitForWebhook hook2
       
       def str2
       str2 = data2.split('&')
       
       def messageStr2
       messageStr2 = str2[0].split('=')
       
       def statusStr2
       statusStr2 = str2[1].split('=')
       
       if( statusStr2[1].equals("success")) { 
           echo "Webhook was called, VM was removed succesfully. Message: ${messageStr2[1]}"
       }else{
           echo "Webhook was called, VM was removed NOT succesfully. Message: ${messageStr2[1]}"
       }

   }
}
