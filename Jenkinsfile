pipeline {
	agent any
	environment {
		scannerHome = tool name: 'sonar_scanner_dotnet', type: 'hudson.plugins.sonar.MsBuildSQRunnerInstallation'
		dockerPort = "${env.BRANCH_NAME == "master" ? 6000 : 6100}"
		kubernetesPort = "${env.BRANCH_NAME == "master" ? 30157 : 30158}"
		project = "NAGP"
		dtr = "jinjinesh"
	}
	options {
	  buildDiscarder logRotator(daysToKeepStr: '10', numToKeepStr: '5')
	  disableConcurrentBuilds()
	  timeout(time: 1, unit: 'HOURS')
	  timestamps()
	}
	stages {
		stage('Checkout') {
			steps {
				checkout scm
			}
		}
		stage('Nuget restore') {
			steps {
				bat "dotnet restore"
			}
		}
		stage('Start sonarqube analysis') {
			when {
				branch 'master'
			}
			steps {
				withSonarQubeEnv('Test_Sonar') {
					echo "dotnet ${scannerHome}/SonarScanner.MSBuild.dll begin /k:${project} /n:${project} /v:1.0"
					bat "dotnet ${scannerHome}/SonarScanner.MSBuild.dll begin /k:${project} /n:${project} /v:1.0"
				}
			}
		}
		stage('Build') {
			steps {
				bat "dotnet build -c Release -o app/build"
			}
		}
		stage('Stop sonarqube analysis') {
			when {
				branch 'master'
			}
			steps {
				withSonarQubeEnv('Test_Sonar') {
					bat "dotnet ${scannerHome}/SonarScanner.MSBuild.dll end"
				}
			}
		}
		stage('Release artificats') {
			steps {
				bat "dotnet publish -c Release -o app/publish"
			}
		}
		stage('Docker image') {
			steps {
				bat "docker build -t ${dtr}/i_jineshjain_${env.branch_name}:${BUILD_NUMBER} --no-cache -f WebApplication4/DockerFile ."
			}
		}
		stage('Containers') {
			parallel {
				stage ('PushtoDTR') {
					steps {
						bat "docker push ${dtr}/i_jineshjain_${env.branch_name}:${BUILD_NUMBER}"
					}
				}
				stage ('PreContainerCheck') {
					steps {
						powershell label: '', script: '''
						$containerRecord = docker ps | Select-String ${env:dockerPort};
						if($containerRecord) {
    						$containerId = $containerRecord.ToString().Split(" ")[0];
    						if($containerId) {
								docker container stop $containerId;
								docker rm $containerId;
							}
						}
						'''
					}
				}
			}
		}
		stage ('Docker deployment') {
			steps {
				bat "docker run -d -p ${dockerPort}:80 --name c_jineshjain_${env.branch_name} ${dtr}/i_jineshjain_${env.branch_name}:${BUILD_NUMBER}"
			}
		}
		stage ('Helm Chart Deployment') {
			steps {
				powershell label: '', script: '''
					$ns = $(kubectl get namespaces | findstr jineshjain-${env:branch_name});
					if(!$ns) {
						kubectl create namespace jineshjain-${env:branch_name};
					}
				'''
				bat "helm upgrade --install web-app-jineshjain-${env.branch_name} helm-chart --set nodePort=${kubernetesPort} --set namespace=jineshjain-${env:branch_name} --set image=${dtr}/i_jineshjain_${env.branch_name}:${BUILD_NUMBER} --set name=web-app-${env.branch_name}"
			}
		}
	}
}