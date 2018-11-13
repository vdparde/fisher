def app
def docker_repo_url = 'docker-repo.skillnetinc.com:8502'
def docker_repo_cred = 'docker.skillnet'
def docker_image_name = 'lll_xstore'
def xstore_container_name = "lll-node-1"
def db_container_name = "lll-node-db-1"
def project_name = "LLL_Base"
def project_env = "DEV"
def xunit_tags = "XUNITDEMO"
def docker_file_directory = '/home/oracle/lll'
def docker_machine_ip="192.168.2.112"

pipeline {

    agent {
        label 'kubehost'
    }


    stages {
    
    	 stage('Clean Up Activity') {
            steps {
                echo 'cleaning pipeline workspace'
                cleanWs()
                echo 'cleaning puppet master directory'
                sh 'sleep 10'

            }
            when {
                expression {
                    return "${params.PIPELINE_ACTIVITY}" == 'deploy';
                }
            }
        }
        
        stage('Copy Installers') {
            steps {
        		
        		
        		checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'dhananjay.patade', url: 'http://192.168.3.69:8888/root/Micros_OMC_16.git']]])	
        		script {
                scannerHome = tool 'sonar101';
    			withSonarQubeEnv('sonar101') {
      			sh "${scannerHome}/bin/sonar-scanner -Dsonar.analysis.mode=publish -Dsonar.host.url=http://sonarqube.skillnetinc.com -Dsonar.projectKey=lululemon -Dsonar.projectName=lululemon -Dsonar.sources=./omc_pos/ -Dsonar.login=1b249939152990fd003a813968afcb44b0bce4c0 -Dsonar.java.binaries=./omc_pos"
    			}
    			
    		   }
        		
        		cleanWs()
        		checkout scm
        		
                copyArtifacts filter: 'workspace/distro-full/OracleRetailXstorePointofService*.zip', fingerprintArtifacts: true, flatten: true, projectName: 'OMC_V16_Test', selector: lastSuccessful(), target: ''
             }
            when {
                expression {
                    return "${params.PIPELINE_ACTIVITY}" == 'deploy';
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
        		
        		withEnv(["docker_file_directory=$docker_file_directory","xstore_container_name=$xstore_container_name","db_container_name=$db_container_name","docker_repo_url=$docker_repo_url","docker_image_name=$docker_image_name","project_name=$project_name","project_env=$project_env","xunit_tags=$xunit_tags","docker_machine_ip=$docker_machine_ip"]) {
		      	sh '''
						DOCKERFILE_DIR=$docker_file_directory
						BASEDIR=$(pwd)
						XSTORE_CONTAINER_NAME=$xstore_container_name
						DB_CONTAINER_NAME=$db_container_name
						
						mv OracleRetailXstorePointofService*.zip xstore.zip 
						rm -rf $BASEDIR/ant.install.properties.mssql
						cp $DOCKERFILE_DIR/ant.install.properties.mssql $BASEDIR/ant.install.properties.mssql
						rm -rf $BASEDIR/xunit.properties
						cp $DOCKERFILE_DIR/xunit.properties $BASEDIR/xunit.properties
						rm -rf ModifedDatabaseScripts
						cp -R $DOCKERFILE_DIR/ModifedDatabaseScripts $BASEDIR/ModifedDatabaseScripts
						
						PRE_CLUSTER_IP=$(cat ant.install.properties.mssql | grep -oh 192.168.* | tail -1)
						
						echo "#########################Starting xstore process#######################"
						echo "########## Clean Up: Removing existing containers ###########################"
						
						sleep 10
						
						echo "########## Starting Database container with name:$DB_CONTAINER_NAME ##############"
						
						if [ CH\$(docker ps -a -f \"name=\$XSTORE_CONTAINER_NAME\" | grep \$XSTORE_CONTAINER_NAME -o) != "CH" ]; then docker rm -f $XSTORE_CONTAINER_NAME; fi
						if [ CH\$(docker ps -a -f \"name=\$DB_CONTAINER_NAME\" | grep linux -o) != "CH" ]; then docker rm -f $DB_CONTAINER_NAME; fi
						
						
						sleep 5
						
						docker run --name=$DB_CONTAINER_NAME -e "ACCEPT_EULA=Y" -e \'SA_PASSWORD=$Killnet123\' -p 1433:1433 -d microsoft/mssql-server-linux:latest
						
						echo "########## Database container is running ####################################"
						
						echo "########## Updating Database IP to installer files"
						sed -i "s/$PRE_CLUSTER_IP/$docker_machine_ip/g" ModifedDatabaseScripts/mssql/make-database.sh
						sed -i "s/$PRE_CLUSTER_IP/$docker_machine_ip/g" ant.install.properties.mssql
						
						echo "########## Creating temp directory for installer files"
						TDIR=$BASEDIR/installertmp
						mkdir -p $TDIR && rm -rf xstore-mssqlserver.tar
						echo "########## Inflating installer ####################"
						unzip -o $BASEDIR/xstore.zip -d $TDIR
						cd $TDIR
						POSDIR=$(dirname "$(find $(pwd) -type d -name *pos*)")
						echo "########## Going to $POSDIR####################"
						cd $POSDIR
						rm -rf resources && mkdir -p resources/db/mssql
						cp $BASEDIR/ModifedDatabaseScripts/mssql/make-database.sh resources/db/mssql
						cp $BASEDIR/ModifedDatabaseScripts/mssql/db-update.sql resources/db/mssql
						zip pos/**install.jar -r resources
						cp $DOCKERFILE_DIR/xstore.install.properties xstore.install.properties
						zip -u pos/**install.jar xstore.install.properties
						rm -rf xstore.install.properties
						tar -cvf $BASEDIR/xstore-mssqlserver.tar pos/
						cd $BASEDIR
						rm -rf $TDIR
						
						
        				
        				rm -rf xstore.zip
        				docker login $docker_repo_url --username=dev1 --password=dev1
        				docker build -t $docker_image_name $BASEDIR
        				docker run -it -d --name=$XSTORE_CONTAINER_NAME  --add-host snsl-vm-7:$docker_machine_ip -e DISPLAY=192.168.1.187:2 -e JAVA_HOME=/usr/orps/java/jdk1.8.0_144 $docker_image_name
						
							
		       		'''
       			}
       		
             }
            when {
                expression {
                    return "${params.PIPELINE_ACTIVITY}" == 'deploy';
                }
            }
        }
 		stage('Dev Testcases execution') {
            steps {
        		withEnv(["docker_file_directory=$docker_file_directory","xstore_container_name=$xstore_container_name","docker_repo_url=$docker_repo_url","docker_image_name=$docker_image_name","project_name=$project_name","project_env=$project_env","xunit_tags=$xunit_tags","additional_email_list=${params.MANAGER_EMAIL_ID}","docker_machine_ip=$docker_machine_ip"]) {
			   
			    sh '''
						docker start $xstore_container_name
						
						docker exec $xstore_container_name rm -rf /tmp/.X11-unix
						
						docker exec $xstore_container_name find /tmp -type f -name .*X*lock -exec rm -rf {} \\;
						
						docker exec $xstore_container_name /bin/bash -c "vncserver && export DISPLAY=192.168.1.187:2"
						
						echo "########## Staring XStore Container: $xstore_container_name ##############################"
						
						docker cp $docker_file_directory/test $xstore_container_name:/home/oracle/xstore/config/
						
						docker cp $docker_file_directory/keys $xstore_container_name:/home/oracle/xstore/res/
						
						docker cp $docker_file_directory/query $xstore_container_name:/home/oracle/xstore/config/test/
						
						echo "######################## Executing test cases on container: $CONTAINER_NAME Project-$PROJECT_NAME Env-$PROJECT_ENV Tags-$XUNIT_TAGS "
						
						docker exec $xstore_container_name /bin/bash -c "export DISPLAY=192.168.1.187:2 && export EMAIL_LIST_EXT=$additional_email_list && cd /home/oracle/Skillnet-XUnit && chmod +x xunit.sh && sleep 5 &&  ./xunit.sh $project_name DEV $xunit_tags"
				'''
	
	   			 }
	   			 
	   			 script {
                    if ("${params.PIPELINE_ACTIVITY}".contains("deploy")) {
                        mail(to: "${params.MANAGER_EMAIL_ID}",
                            subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
                            body: "Please go to ${env.BUILD_URL}input/.");
                        input message: 'Test Cases Completed (Developer Test Cases), Deploy this build in Dev environment?', ok: 'Proceed'
                    }
                }
 		
             }
             when {
                expression {
                  return "${params.ENVIRONMENT}".contains("dev")
                }
            }
           
        }
         stage('Copy to Puppet Master') {
            steps {
                parallel 'Copy XStore Installer': {

                        echo 'Copying xstore installer'
                        sh 'sleep 2'

                    },
                    failFast: true


            }
            when {
                expression {
                    return "${params.PIPELINE_ACTIVITY}" == 'deploy';
                }
            }

        }
         stage('DEV Deployment') {
            steps {
                echo 'Deploy Oracle Db'
                echo 'Deploy Java'
                echo 'Deploy XStore in DEV environment'
               
                sh 'sleep 4'
                
                script {
                    if ("${params.ENVIRONMENT}".contains("sit")) {
                        mail(to: "${params.MANAGER_EMAIL_ID}",
                            subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
                            body: "Please go to ${env.BUILD_URL}input/.");
                        input message: 'Deployment Completed In Dev environment, Do you want to move to next stage?', ok: 'Proceed'
                    }
                }
            }
            when {
                expression {
                    return "${params.ENVIRONMENT}".contains("dev") && "${params.PIPELINE_ACTIVITY}" == 'deploy';
                }
            }
        }
        stage('SIT Testcases execution') {
            steps {
        		withEnv(["docker_file_directory=$docker_file_directory","xstore_container_name=$xstore_container_name","docker_repo_url=$docker_repo_url","docker_image_name=$docker_image_name","project_name=$project_name","project_env=$project_env","xunit_tags=$xunit_tags","additional_email_list=${params.MANAGER_EMAIL_ID}","docker_machine_ip=$docker_machine_ip"]) {
			   
			    sh '''
						docker start $xstore_container_name
						
						docker exec $xstore_container_name rm -rf /tmp/.X11-unix
						
						docker exec $xstore_container_name find /tmp -type f -name .*X*lock -exec rm -rf {} \\;
						
						docker exec $xstore_container_name /bin/bash -c "vncserver && export DISPLAY=192.168.1.187:2"
						
						echo "########## Staring XStore Container: $xstore_container_name ##############################"
						
						docker cp $docker_file_directory/test $xstore_container_name:/home/oracle/xstore/config/
						
						docker cp $docker_file_directory/keys $xstore_container_name:/home/oracle/xstore/res/
						
						docker cp $docker_file_directory/query $xstore_container_name:/home/oracle/xstore/config/test/
						
						echo "######################## Executing test cases on container: $CONTAINER_NAME Project-$PROJECT_NAME Env-$PROJECT_ENV Tags-$XUNIT_TAGS "
						
						docker exec $xstore_container_name /bin/bash -c "export DISPLAY=192.168.1.187:2 && export EMAIL_LIST_EXT=$additional_email_list &&  cd /home/oracle/Skillnet-XUnit && chmod +x xunit.sh && sleep 5 && ./xunit.sh $project_name SIT $xunit_tags"
				'''
	
	   			 }
	   			 
	   			 script {
                    if ("${params.PIPELINE_ACTIVITY}".contains("deploy")) {
                        mail(to: "${params.MANAGER_EMAIL_ID}",
                            subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
                            body: "Please go to ${env.BUILD_URL}input/.");
                        input message: 'Automated Test Cases(SIT test cases) Completed, Deploy this build in SIT environment?', ok: 'Proceed'
                    }
                }
 		
             }
             when {
                expression {
                  return "${params.ENVIRONMENT}".contains("sit")
                }
            }
           
        }
        stage('SIT Deployment') {
            steps {
                echo 'Deploy Oracle Db'
                echo 'Deploy Java'
                echo 'Deploy XStore in SIT environment'
               
                sh 'sleep 4'
                
                script {
                    if ("${params.ENVIRONMENT}".contains("uat")) {
                        mail(to: "${params.MANAGER_EMAIL_ID}",
                            subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
                            body: "Please go to ${env.BUILD_URL}input/.");
                        input message: 'Deployment Completed In SIT environment, Do you want to move to next stage?', ok: 'Proceed'
                    }
                }
            }
            when {
                expression {
                    return "${params.ENVIRONMENT}".contains("sit") && "${params.PIPELINE_ACTIVITY}" == 'deploy';
                }
            }
        }
        stage('UAT Testcases execution') {
            steps {
                echo 'Checking out uat test cases'

                echo 'Cleaning up previous test cases'
                echo 'Copying Test cases'
                echo 'Started Automated Testing'
               script {
                    if ("${params.PIPELINE_ACTIVITY}".contains("deploy")) {
                        mail(to: "${params.MANAGER_EMAIL_ID}",
                            subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
                            body: "Please go to ${env.BUILD_URL}input/.");
                        input message: 'Automated Test Cases(UAT test cases) Completed, Deploy this build in UAT environment?', ok: 'Proceed'
                    }
                }
            }
            when {
                expression {
                    return "${params.ENVIRONMENT}".contains("uat");
                }
            }
        }
        stage('UAT Deployment') {

            steps {
                echo 'Deploy Oracle Db'
                echo 'Deploy Java'
                echo 'Deploy XStore in UAT environment'
                echo 'Deploy TAF in UAT environment'
                sleep 2
                
            }
            when {
                expression {
                    return "${params.ENVIRONMENT}".contains("uat") && "${params.PIPELINE_ACTIVITY}" == 'deploy';
                }
            }
        }
        
        stage('Copy to Production Puppet Master') {
            steps {
                mail(to: "${params.MANAGER_EMAIL_ID}",
                    subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
                    body: "Please go to ${env.BUILD_URL}input/.");
                input message: 'Completed previous stages, Copy Installers to Production puppet master?', ok: 'Approved'
                echo 'Copied Installers to Puppet Master'
                sleep 2
            }
            when {
                expression {
                    return "${params.PIPELINE_ACTIVITY}".contains("deploy");
                }
            }

        }

	}
    parameters {
        string(defaultValue: 'dev', description: '*Comma separated A) dev - Development environment </br> B) sit - SIT environment </br> C) uat - UAT environment', name: 'ENVIRONMENT')
        string(defaultValue: 'test', description: '*Comma separated A) deploy - Deploy & execute test cases </br> B) test - Do not deploy only execute test cases </br>this is applicable for all environment', name: 'PIPELINE_ACTIVITY')
        string(defaultValue: 'dhananjay.patade@skillnetinc.com', description: 'Email id of manager(s) who will approve pipeline stages (Comma separated values)', name: 'MANAGER_EMAIL_ID')
        string(defaultValue: '33', description: 'Build no of XStore installer to be installed', name: 'XSTORE_BUILD_NUMBER')
        string(defaultValue: '192.168.2.83', description: 'IP of puppet master', name: 'PUPPET_MASTER_IP')
        string(defaultValue: 'XUNITDEMO', description: 'XUNIT tags to be tested', name: 'ENABLED_TAGS')
    }

}
