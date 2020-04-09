node{
	stage ("Get tools version") {
		Maven_Version = sh (script: 'ls /var/jenkins_home/tools/hudson.tasks.Maven_MavenInstallation/', returnStdout: true).trim()
		JAVA_Version = sh (script: 'ls /var/jenkins_home/tools/hudson.model.JDK/', returnStdout: true).trim()
	}
}

pipeline {
agent any
	parameters {
		choice(name: 'MavenVersion', choices: "${Maven_Version}", description: 'On this step you need select Maven Version')
		choice(name: 'JavaVersion', choices: "${JAVA_Version}", description: 'On this step you need select JAVA Version')
		choice(name: 'TomcatVersion', choices: "tomcat-8\ntomcat-7", description: 'Please select ENV server for deploy you APP')
		choice(name: 'Deploing', choices: "NO\nYES", description: 'Option for allow/decline deploy to ENV')
		choice(name: 'ENVIRONMENT', choices: "TEST\nSTAGE\nPROD", description: 'Please select ENV server for deploy you APP')
		text(name: 'branch_name', defaultValue: 'master', description: 'Please fill branch/tag (default value is master)')
	}
	environment {
		STAGE = 'stage.projects.local'
		TEST = 'test.projects.local'
		PROD = 'prod.projects.local'
		git_url = 'https://git.projects.local/test_git.git'
		git_cred_id = '3dd2323-avf43s-40cb-4sdfght-s44s0993s09'
		docker_registry = 'projectname.projects.local:5000'
		APP_EXTPORT = '30100'
		service_network = "docker_app_net"
		Maven_OPTS = '-Dmaven.test.failure.ignore'
		artifactory_user = 'publisheruser'
		artifactory_password = 'strongpassword'
		artifactory_url = 'http://projectname.projects.local/artifactory'
		artifactory_target_folder = 'tomcat'
	}
	tools {
		maven "${params.MavenVersion}"
		jdk "${params.JavaVersion}"
	}
stages {
	stage('Source Checkout and switch to tag if exist') {
		steps {
		    git credentialsId: '${git_cred_id}', url: '${git_url}'
            sh 'git checkout ${branch_name}'
		}
	}
	stage('Build With maven') {
		steps {
			script {
			    sh "mvn $Maven_OPTS clean install -U"
				sh 'cp target/*.jar app.jar'
			}
		}
	}
	stage('Creating dockerfile & docker-compose files') {
		steps {
			script {
				sh """
				echo "version: '3'
services:
  app:
    container_name: ${JOB_NAME}_app
    image: ${docker_registry}/${JOB_NAME}:v${BUILD_NUMBER}
    restart: always
    ports:
      - "${APP_EXTPORT}:8080"
    environment:
      - KEY1=VALUE1
      - KEY2=VALUE2
      - KEY3=VALUE3
      - KEY4=VALUE4
    volumes:
      - ./persistent/storage1:/opt/data1
      - ./persistent/storage1/config.yaml:/opt/config/config.yaml">docker-compose.yaml
				"""
				
				sh """
				echo "
FROM centos:7
RUN yum install -y java-1.${JavaVersion}.0-openjdk java-1.${JavaVersion}.0-openjdk-devel wget
WORKDIR /opt/
RUN wget ${artifactory_url}/tomcat/apache-${TomcatVersion}.tar.gz
RUN tar -xf /opt/apache-${TomcatVersion}.tar.gz -C /opt/
RUN rm -rf /opt/apache-${TomcatVersion}.tar.gz
RUN rm -rf /opt/apache-tomcat/webapps/*
ENV JAVA_HOME /usr/lib/jvm/java-1.${JavaVersion}.0-openjdk/
RUN export JAVA_HOME
COPY app.jar /opt/apache-tomcat/webapps/
RUN mkdir -p /usr/share/fonts/ && cd /usr/share/fonts/
RUN wget -r -nH --cut-dirs=2 --no-parent http://projectname.projects.local/artifactory/fonts/
RUN fc-cache -fv /usr/share/fonts/
EXPOSE 8080
CMD /opt/apache-tomcat/bin/catalina.sh run">Dockerfile
				"""
			}
		}
	}
	stage('push_to_Artifactory') {
		steps {
			script{
				rtServer (
					id: 'Artifactory-1',
					url: "${artifactory_url}",
					username: "${artifactory_user}",
					password: "${artifactory_password}"
					)
				rtUpload (
					serverId: 'Artifactory-1',
					spec: '''{
					"files": [
						{
						"pattern": "target/*.jar",
						"target": "${artifactory_target_folder}/"
						}
					]
					}''',
					buildName: "${JOB_NAME}",
					buildNumber: "${BUILD_NUMBER}")
			}
		}	
	}
	stage('Building image') {
		steps{
			script {
				sh "docker build . --network ${service_network} -t ${docker_registry}/${JOB_NAME}:v${BUILD_NUMBER}"
				sh "docker login https://${docker_registry}"
				sh "docker push ${docker_registry}/${JOB_NAME}:v${BUILD_NUMBER}"
				sh "docker tag ${docker_registry}/${JOB_NAME}:v${BUILD_NUMBER} ${docker_registry}/${JOB_NAME}:latest"
				sh "docker push ${docker_registry}/${JOB_NAME}:latest"
			}
		}
	}
	stage('Deploing image to ENV') {
		when { 
				expression { params.Deploing == 'YES' }
		}
		steps {
			sh "scp -o StrictHostKeyChecking=no ./docker-compose.yaml root@${env."${params.ENVIRONMENT}"}:/root/"
			sh "ssh -o StrictHostKeyChecking=no root@${env."${params.ENVIRONMENT}"} 'docker-compose up --build -d'"
		}
	}
	stage('Remove Unused docker image') {
		steps{
			sh "docker rmi -f ${docker_registry}/${JOB_NAME}:latest"
			sh "docker rmi -f ${docker_registry}/${JOB_NAME}:v${BUILD_NUMBER}"
		}
	}
}
	post {
		always {
			echo 'One way or another, I have finished'
			deleteDir() /* clean up our workspace */
		}
	}
}
