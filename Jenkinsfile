node {
    def mvnHome
    mvnHome = tool 'mvn'
    env.PATH = "${mvnHome}/bin:${env.PATH}"
    
    def vmName
    vmName = "Debian-Runner-${BUILD_NUMBER}-TJ"
       
    def ip
    ip = "172.16.20.159"
    
    stage('Preparation') {
        //fetch git repo, and get new files
        git url:'https://github.com/hfo/spring-petclinic.git'
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
    
   //checkpoint 'before Create VM & Deploy App'
    
   stage('Create VM & Deploy App'){
            // in this array we'll place the jobs that we wish to run
       
       def parallelExecutions = [:]

            //running the job 4 times concurrently
            //the dummy parameter is for preventing mutation of the parameter before the execution of the closure.
            //we have to assign it outside the closure or it will run the job multiple times with the same parameter "4"
            //and jenkins will unite them into a single run of the job

        parallelExecutions["exec1"] = {
            //Parameters:
            //param1 : an example string parameter for the triggered job.
            //dummy: a parameter used to prevent triggering the job with the same parameters value.
            //       this parameter has to accept a different value each time the job is triggered.
            build job: 'deploy_VM_vSphere_Job', parameters: [[$class: 'StringParameterValue', name: 'ip', value: ip],[$class: 'StringParameterValue', name: 'vmName', value: vmName]]
        }
       
       parallelExecutions["exec2"] = {
           // On new Azure VM 
           // configure VM, upload IP, get Petclinic and deploy Petclinic
            build job: 'Azure_pipe'
        }
    parallel parallelExecutions

       //sh 'ssh administrator@172.16.20.93 "rm -f petclinic-1.0.0.jar; wget http://172.16.20.92:8081/repository/Jenkins-Repo/de/proficom/cdp/petclinic/1.0.0/petclinic-1.0.0.jar; ls"'
       //sh 'ssh administrator@172.16.20.93 "nohup java -jar petclinic-1.0.0.jar &"'
       
       //input id: 'Wait-for-manual-continue-1', message: 'Waiting for manual continue'        
   }
    
   stage('Functional tests'){
       //build job: 'LeanFT_ALM_Job'
       //build job: 'UFT_Job'
   }
    
   stage('Performance tests'){
       
       build job: 'Loadrunner_Job_mini'
       build job: 'PerformanceCenter_Job'   
   }
    
   stage('Clean up testenvironment'){
       //input id: 'Wait-for-manual-continue-2', message: 'Waiting for manual continue' 
       
       //synchronize jenkins with oo-flow
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
    stage('Deploy to Prod'){
        build job: 'oo_deploy_petclinic_via_csa', parameters: [[$class: 'StringParameterValue', name: 'pipelineBuildNumber', value: BUILD_NUMBER]]
    }
}
