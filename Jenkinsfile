node {
    def mvnHome
    mvnHome = tool 'mvn'
    env.PATH = "${mvnHome}/bin:${env.PATH}"
    
    def vmName
    vmName = "Debian-Runner-${BUILD_NUMBER}-TJ"
    
    //this ip is used for Test-VM
    def ip
    ip = "172.16.20.159"
    
    //this ip is used for Prod-VM
    def prod_ip
    prod_ip = "172.16.20.92"
    
    stage('Preparation') {
        //fetch git repo, and get new files
        git url:'https://github.com/cipShowcasePC/spring-petclinic.git'
    }
    
    stage('Build&Upload Maven Artifact(jar)') {
        //maven build process
        sh './mvnw clean install -Dmaven.test.skip=true'
        
        //sonarqube static code analysis
        sh "./mvnw sonar:sonar -Dsonar.host.url=http://172.16.20.157:9000"
        
        //upload built jar to repo for further availibility 
        build job: 'nexus_uploader_job'
    }
    
    
    stage('Build&Upload Docker Image'){
        //remove dockerfile and jar from system to asssure new versions will get downloaded to system
        sh 'rm -f Dockerfile'
        sh 'rm -f petclinic-1.0.0.jar'
        
        //get new versions of dockerfile and jar from repository
        sh'wget http://172.16.20.157:8081/repository/Jenkins-Repo/Dockerfile'
        sh'wget http://172.16.20.157:8081/repository/Jenkins-Repo/de/proficom/cdp/petclinic/1.0.0/petclinic-1.0.0.jar'
        
        //build docker image with tag petclinic_alpine
        sh 'docker build -t petclinic_alpine .'
        
        //retag image to relate it to local repo | push it to the repository
        sh 'docker tag petclinic_alpine 172.16.20.157:8082/petclinic_alpine'
        sh 'docker push 172.16.20.157:8082/petclinic_alpine'
        
        //retag image to relate it to dockerhub repo | push it to the repository
        sh 'docker login -u cipshowcasepc -p password'
        sh 'docker tag petclinic_alpine cipshowcasepc/petclinic-docker-images'
        sh 'docker push cipshowcasepc/petclinic-docker-images'
   }
    
   stage('Create VM & Deploy App'){
        
       // at this point the pipeline will split into 3 parallel executions 
       def parallelExecutions = [:]

       parallelExecutions["exec1"] = {
            //parallel execution branch 1, which will create a vSphere VM, configure it and deploy the petclinic via docker afterwards
            build job: 'deploy_VM_vSphere_Job', parameters: [[$class: 'StringParameterValue', name: 'ip', value: ip],[$class: 'StringParameterValue', name: 'vmName', value: vmName]]
       }
       
       parallelExecutions["exec2"] = {
           // parallel execution branch 2
           // On new Azure VM 
           // configure VM, upload IP, get Petclinic and deploy Petclinic
            build job: 'Azure_pipe'
        }
       parallelExecutions["exec3"] = {
           // parallel execution branch 3
           // On new Azure VM 
           // install Selenium on port 4444
            build job: 'Azure_Selenium'
        }
    parallel parallelExecutions
       
   }
   
   // this input can be uncommented to make the pipeline stop at this point of execution and only continue after human input
   // input id: 'Wait-for-manual-continue-1', message: 'Test-VM have been created, continoue with tests after manual continue' 
    
   stage('Functional tests'){
        sleep time: 60, unit: 'SECONDS' 
       //commented due to problems with license of uft
       //build job: 'LeanFT_ALM_Job', parameters: [[$class: 'StringParameterValue', name: 'vmName', value: vmName]]
       //build job: 'UFT_Job', parameters: [[$class: 'StringParameterValue', name: 'vmName', value: vmName]]
       //build job: 'BPT_ALM_Test', parameters: [[$class: 'StringParameterValue', name: 'vmName', value: vmName]]
   }
    
   stage('Performance tests'){
       build job: 'Loadrunner_Job_mini', parameters: [[$class: 'StringParameterValue', name: 'vmName', value: vmName]]
       build job: 'PerformanceCenter_Job', parameters: [[$class: 'StringParameterValue', name: 'vmName', value: vmName]]   
   }
    
   stage('Clean up testenvironment'){
       //this input can be uncommented to make the pipeline stop at this point of execution and only continue after human input
       //input id: 'Wait-for-manual-continue-2', message: 'Waiting for manual continue' 
       
       //to synchronize jenkins with oo-flow after processing the automated executions webhooks will be used
       def hook2
       hook2 = registerWebhook()
        
       // job will be called handing over the webhook's url so the oo-flow is able to do the callback operation
       build job: 'oo_remove_runner_vm', parameters: [[$class: 'StringParameterValue', name: 'hook_url', value: hook2.getURL()],[$class: 'StringParameterValue', name: 'vmName', value: vmName],[$class: 'StringParameterValue', name: 'pipelineBuildNumber', value: BUILD_NUMBER]]
       
       // console output to show pipeline ios wating for webhook to be called
       echo "Waiting for POST to ${hook2.getURL()}"
       
       // analysing the return values within the http-Request
       // data is submitted in the form: "messageName=messageText&statusname=statusText"
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
    stage('Deploy to Prod'){
        //calling job that will call oo flow that will deploy application to "prod machines"
        build job: 'oo_deploy_petclinic_via_csa', parameters: [[$class: 'StringParameterValue', name: 'pipelineNumber', value: BUILD_NUMBER],[$class: 'StringParameterValue', name: 'vmIp', value: prod_ip]]
    }
}
