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
		choice(name: 'Deploing', choices: "YES\nNO", description: 'Option for allow/decline deploy to ENV')
		choice(name: 'ENVIRONMENT', choices: "TEST\nSTAGE\nPROD", description: 'Please select ENV server for deploy you APP')
		choice(name: 'TomcatVersion', choices: "tomcat-7\ntomcat-8\ntomcat-9", description: 'Please select ENV server for deploy you APP')

	}
	environment {
		STAGE = '192.168.23.7'
		TEST = '192.168.23.7'
		PROD = '192.168.23.7'
		registry = 'registry.domain.com:5000'
		APP_EXTPORT = '30100'
		GIT_SOURCE = "https://github.com/werdervg/${JOB_NAME}.git"
//		replace_registry_path='$registry/$JOB_NAME:v$BUILD_NUMBER'
		service_network = "root_app-network"
		Maven_OPTS = '-Dmaven.test.failure.ignore'
		artifactory_user = 'publisheruser'
		artifactory_password = 'pa@sswo2rd1'
		artifactory_url = 'http://artifactory:8081/artifactory'
	}
	tools {
		maven "${params.MavenVersion}"
		jdk "${params.JavaVersion}"
	}
stages {
	stage('Version Java&Tomcat compatibility check') {
		when { 
			anyOf {
				expression { params.JavaVersion == '7' && params.TomcatVersion == 'tomcat-9' };
				expression { params.JavaVersion == '8' && params.TomcatVersion == 'tomcat-7' };
				expression { params.JavaVersion == '9' && params.TomcatVersion == 'tomcat-7' }
			}	
		}
		steps {
			sh 'echo "ERROR: ${TomcatVersion} does not support ${JavaVersion}" && exit 1'
		}
	}
	stage('Get Source Code && Build With maven') {
		steps {
			script {
				sh "git clone $GIT_SOURCE"
				sh """
				echo "version: '3'
services:
  app:
    container_name: JOB_NAME_app
    #image: REGISTRY_NAME/JOB_NAME:vBUILD_NUMBER
    #build: .
    restart: always
    ports:
      - "APP_EXTPORT:8080"
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
EXPOSE 8080
CMD /opt/apache-tomcat/bin/catalina.sh run">Dockerfile
				"""
				
				sh "cp -r ./$JOB_NAME/* ./"
//				sh "mv Dockerfile_${JavaVersion}_${TomcatVersion} Dockerfile"
				sh "mvn $Maven_OPTS clean package"
				sh 'cp target/*.jar app.jar'
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
						"target": "$JOB_NAME/"
						}
					]
					}''',
					buildName: "$JOB_NAME",
					buildNumber: "$BUILD_NUMBER")
			}
		}	
	}
	stage('Building image and preparing compose file') {
		steps{
			script {
				sh "sed -i s/#build/build/g docker-compose.yaml"
				sh "sed -i s/JOB_NAME/${JOB_NAME}/g docker-compose.yaml"
				sh "sed -i s/APP_NAME/${JOB_NAME}/g docker-compose.yaml"
				sh "sed -i s/APP_EXTPORT/${APP_EXTPORT}/g docker-compose.yaml"
				sh "sed -i s/REGISTRY_NAME/${registry}/g docker-compose.yaml"
				sh "sed -i s/BUILD_NUMBER/${BUILD_NUMBER}/g docker-compose.yaml"
				sh "docker build . --network ${service_network} -t $registry/$JOB_NAME:v$BUILD_NUMBER"
				sh "docker login https://$registry"
				sh "docker push $registry/$JOB_NAME:v$BUILD_NUMBER"
				sh "docker tag $registry/$JOB_NAME:v$BUILD_NUMBER $registry/$JOB_NAME:latest"
				sh "docker push $registry/$JOB_NAME:latest"
				sh "sed -i s/build/#build/g docker-compose.yaml"
				sh "sed -i s/#image/image/g docker-compose.yaml"
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
			sh "docker rmi -f $registry/$JOB_NAME:latest"
			sh "docker rmi -f $registry/$JOB_NAME:v$BUILD_NUMBER"
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
