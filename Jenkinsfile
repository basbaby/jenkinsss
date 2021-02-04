pipeline {
    agent any
    tools {
        maven "Maven" 
	  
	    
    }
   stages {
   	stage("SonarQube Analysis"){
        	steps {
                	withSonarQubeEnv("Sonarqube") {
                    		sh "mvn -f pom.xml sonar:sonar -Dsonar.sources=src/"
                    		script {
		    			LAST_STARTED = env.STAGE_NAME
					container_Up = false
                    			timeout(time: 1, unit: "HOURS") { 
                        			sh "curl -u admin:admin -X GET -H \"Accept: application/json\" http://104.248.169.167:9000/api/qualitygates/project_status?projectKey=com.mycompany:jenkinsss > status.json"
                        			def json = readJSON file:"status.json"
                        			echo "${json.projectStatus}"
                        			if ("${json.projectStatus.status}" != "OK") {
                            				currentBuild.result = "FAILURE"
                           				error("Pipeline aborted due to quality gate failure.")
                           			}
                        		}     
                    		}
                	}
                }
	}
       /*stage("upload to nexus") {
      steps {
        script {
	  LAST_STARTED = env.STAGE_NAME
          echo "hello";
          pom = readMavenPom file: "pom.xml";
          filesbyGlob = findFiles(glob: "target/*.jar");
          echo "${filesbyGlob[0].path}";
          nexusArtifactUploader(artifacts: [[artifactId: pom.artifactId, classifier: "", file: filesbyGlob[0].path, type: "jar"]], credentialsId: "nexus", groupId: pom.groupId, nexusUrl: "localhost:8081", nexusVersion: "nexus3", protocol: "http", repository: "com.testnjc", version: currentBuild.number.toString())
        }
      }
    }*/
	   
      stage("Build") {
      		steps {
	    		script {
				LAST_STARTED = env.STAGE_NAME
				container_Up = false
				configFileProvider([configFile(fileId: "706c4f0b-71dc-46f3-9542-b959e2d26ce7", variable: "settings")]){
				sh "mvn -f pom.xml -s $settings clean install -DskipTests "     
				}
		    	} 
            	}    
      } 
	   
      stage("Build image") {
      		steps {
        		script {
			  	//sh "docker stop apiops-anypoint-jenkins-sapi" 
        		  	//sh "docker rm apiops-anypoint-jenkins-sapi"
			   	LAST_STARTED = env.STAGE_NAME
				container_Up = false
			   	sh "docker build -t jenkinsss:mule -f Dockerfile ."
                	 
                        }
               }
      }

      stage("Run container") {
      		steps {
        		script {
			     	LAST_STARTED = env.STAGE_NAME
          		    	sh " docker run -itd -p 8082:8081 --name jenkinsss jenkinsss:mule"
				container_Up = true
				sh "sleep 60"
       			}
		}
     }
   	
     stage ("Munit Test"){
        	steps {
			script {
			   	LAST_STARTED = env.STAGE_NAME
				configFileProvider([configFile(fileId: "706c4f0b-71dc-46f3-9542-b959e2d26ce7", variable: "settings")]){
			   	sh "mvn -f pom.xml -Dhttp.port=8086 -s $settings -Dkey=mymulesoft test"
				
				publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: "jenkinsss/target/site/munit/coverage", reportFiles: "summary.html", reportName: "Munit coverage Report", reportTitles: ""])
				}
			}		
        	}    
     }
     stage("Functional Testing"){
        	steps {
			script {
				LAST_STARTED = env.STAGE_NAME
				sh "mvn -f cucumber-API-Framework/pom.xml test -Dtestfile=cucumber-API-Framework/src/test/javarunner.TestRunner.java"
				cucumber(failedFeaturesNumber: -1, failedScenariosNumber: -1, failedStepsNumber: -1, fileIncludePattern: "cucumber.json", jsonReportDirectory: "cucumber-API-Framework/target", pendingStepsNumber: -1, skippedStepsNumber: -1, sortingMethod: "ALPHABETICAL", undefinedStepsNumber: -1)
			}
        		
             	}   
     }
	
	  stage ("Jmeter Testing"){
	    steps{
		    sh "mvn -f apipos-jmeter-automation-master/pom.xml clean verify -Djmeter.path=/opt/jmeter/5.3/libexec/bin/"
		    perfReport filterRegex: "", sourceDataFiles: "**/*.jtl"
		    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: "", reportFiles: "**/index.html", reportName: "Jmeter- performance test report", reportTitles: "Jmeter Reports"])
	
	    	   }
	   }
	   
    //stage("Generate Reports") {
    //		steps {
//			script {
//				LAST_STARTED = env.STAGE_NAME
//				cucumber(failedFeaturesNumber: -1, failedScenariosNumber: -1, failedStepsNumber: -1, fileIncludePattern: "cucumber.json", jsonReportDirectory: "cucumber-API-Framework/target", pendingStepsNumber: -1, skippedStepsNumber: -1, sortingMethod: "ALPHABETICAL", undefinedStepsNumber: -1)
//			}
 //               }
 //   }
	   
    stage("Archetype"){
        	steps {
			script {
		    		LAST_STARTED = env.STAGE_NAME
		    		configFileProvider([configFile(fileId: "706c4f0b-71dc-46f3-9542-b959e2d26ce7", variable: "settings")]){
					def data = "archetype.artifactId=generic-db-sapi"
                  		        writeFile(file: "archetype.properties", text: data)
                    			sh "mvn -f pom.xml -s $settings archetype:create-from-project -Darchetype.properties=/var/lib/jenkins/workspace/devops-project/archetype.properties"
		    			sh "mvn -f jenkinsss/target/generated-sources/archetype/pom.xml -s $settings clean deploy -DaltSnapshotDeploymentRepository=nexus-snapshots::http://104.248.169.167:8081/repository/maven-snapshots/"
                  		} 
			}
              }   
     }
	  
    stage("Deploy to Cloudhub"){
        	steps {
			script {
				LAST_STARTED = env.STAGE_NAME
				configFileProvider([configFile(fileId: "706c4f0b-71dc-46f3-9542-b959e2d26ce7", variable: "settings")]){
					
				sh "mvn -f pom.xml -s $settings package deploy -DmuleDeploy -DskipTests -Dkey=mymulesoft -Danypoint.username=njcdemo1 -Danypoint.password=Njclabs@123 -DapplicationName=jenkinsss-mule -Dcloudhub.region=us-east-2 -Danypoint.platform.client_id=2873c31e7152405e9dc38600007108e8 -Danypoint.platform.client_secret=70da4Da4228749B7A33C848E9a3C0849"
				}
			}
             	}
    }
	   
    stage("Email") {
      		steps {
			script {
          		    	readProps= readProperties file: "cucumber-API-Framework/email.properties"
          		    	echo "${readProps["email.to"]}"
        		    	emailext(subject: "$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!", body: "Build is Success.Please find the functional testing reports. In order to check the logs, please go to url: $BUILD_URL"+readFile("jenkinsss/emailTemplate.html"), attachmentsPattern: "cucumber-API-Framework/target/cucumber-reports/report.html", from: "${readProps["email.from"]}", mimeType: "${readProps["email.mimeType"]}", to: "${readProps["email.to"]}")
                        }
		}
    }   
    stage("Kill container") {
      		steps {
        		script {
	  	        	LAST_STARTED = env.STAGE_NAME		
          		    	sh "docker rm -f jenkinsss"
				sh "sleep 10"
				sh "docker rmi jenkinsss:mule"
				
        		}
      		}
    	} 
   
   }
   post {
        failure {
	    script {
	    		emailbody = "Build Failed at $LAST_STARTED Stage. Please find the attached logs for more details."
          		readProps= readProperties file: "cucumber-API-Framework/email.properties"
				emailext(subject: "$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!", body: "$emailbody", attachLog: true, from: "${readProps["email.from"]}", to: "${readProps["email.to"]}")
                        if (container_Up)  {
		 		sh "docker rm -f jenkinsss"	
				sh "sleep 10"
				sh "docker rmi jenkinsss:mule"
				
		 }
	    } 
        }
    } 
}
