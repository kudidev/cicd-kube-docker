def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

pipeline {
	agent any
	tools{
		maven "MAVEN3"
		jdk "oracleJDK11"
	}
	
	environment {
        registry = "kudidev/vproappdock"
        registryCredential = "dockerhub"
    }
	

	
	stages{

		stage('BUILD'){
			steps {
				sh 'mvn clean install -DskipTests'
			}
			
			post  {
				success {
					echo 'Now Archiving..'
					archiveArtifacts artifacts: '**/target/*.war'
				}
			}
		}
		
		stage('UNIT TEST'){
			steps{
				sh 'mvn test' 
			}
		
		}
		
		stage('Checkstyle Analysis'){
			steps{
				sh 'mvn checkstyle:checkstyle' 
			}
            
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }		
		}

        stage ("Build App Image"){
			steps {
				script {
					dockerImage = docker.build registry + ":V-$BUILD_NUMBER"
				}
			}
		}

        stage ("Upload Image to DockerHub"){
			steps {
				script {
					docker.withRegistry('',registryCredential){
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push('latest')

                    }
				}
			}
		}
        stage ("Remove Unused Docker"){
			steps {
					sh" docker rmi $registry:V$BUILD_NUMBER"
			}
		}

		stage ('Sonar Analysis'){
			environment {
				scannerHome = tool 'sonar4.7'    
				
			}
			
			steps {
				withSonarQubeEnv('Sonarqube'){
					 sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
					-Dsonar.projectName=vprofile \
					-Dsonar.projectVersion=1.0 \
					-Dsonar.sources=src/ \
					-Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
					-Dsonar.junit.reportsPath=target/surefire-reports/ \
					-Dsonar.jacoco.reportsPath=target/jacoco.exec \
					-Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
				}	
			}
		}
		
		stage ("Quality Gate"){
			steps {
				timeout(time: 1, unit: 'HOURS') {
					waitForQualityGate abortPipeline: true
				
				}
			
			}
		}

        stage ("Kubernetes Deploy"){
            agent{label 'KOPS'}
                steps { 
					    sh" helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${$BUILD_NUMBER}) --namespace prod"
			}
		}
		
	}
	post {
      always {
        echo 'Slack Notifications.'
			slackSend channel: '#jenkinscicd',
            color: COLOR_MAP[currentBuild.currentResult],
             message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}