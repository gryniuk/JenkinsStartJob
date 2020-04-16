pipeline {
agent any
	environment {
		STAGE = 'someproject.proj.local'
		STAGEDB = 'db-stage.proj.local'
		STAGE_home_domain = 'https://stage-URL'
		
		TEST = 'someproject.proj.local'
		TESTDB = 'db-test.proj.local'
		TEST_home_domain = 'https://test-URL'
		
		PROD = 'someproject.proj.local'
		PRODDB = 'db-prod.proj.local'
		PROD_home_domain = 'https://PROD-URL'
		
		docker_registry = 'someproject.proj.local:5000'
		
		git_url = 'https://git.proj.local/someproject.git'
		git_cred_id = 'b01ac2e6-ba0d-40cb-834b-9b0a883f9600' 
		
		APP_EXTPORT = '30100'
		service_network = "docker_app_net"
		
		Maven_OPTS = 'clean install -P production'
		MAVEN_SETTINGS_XML_ID = 'e7c71e35-454f-4bb8-a942-ae83bd5a1caf'
		
        artifactory_ID= 'artifactory'
		artifactory_target_folder = 'test-repo/'
	}
	parameters {
		choice(name: 'MavenVersion', choices: "Maven 3.3.3", description: 'On this step you need select Maven Version')
		choice(name: 'JavaVersion', choices: "8\n7", description: 'On this step you need select JAVA Version')
		choice(name: 'TomcatVersion', choices: "tomcat-8\ntomcat-7", description: 'Please select ENV server for deploy you APP')
		choice(name: 'Deploing', choices: "NO\nYES", description: 'Option for allow/decline deploy to ENV')
		choice(name: 'ENVIRONMENT', choices: "TEST\nSTAGE\nPROD", description: 'Please select ENV server for deploy you APP')
	}
	tools {
		maven "${params.MavenVersion}"
		jdk "${params.JavaVersion}"
	}
stages {
	stage('Source Checkout and switch to tag if exist') {
		steps {
            git credentialsId: "${git_cred_id}", url: "${git_url}"
            sh 'git checkout $(git describe --tags `git rev-list --tags --max-count=1`)'
            script {
                env.branch_name = sh( script: 'git describe --tags `git rev-list --tags --max-count=1`', returnStdout: true).trim()
            }
        }
	}
	stage('Build With maven') {
		steps {
             configFileProvider([configFile(fileId: "${MAVEN_SETTINGS_XML_ID}", variable: 'MAVEN_SETTINGS_XML')]) {
                sh 'mvn -s $MAVEN_SETTINGS_XML $Maven_OPTS'
                }
                sh 'cp target/*.war ./'
		}
	}
	stage('Creating dockerfile & docker-compose files') {
		steps {
			script {
				sh """
				echo "db.host=${env."${params.ENVIRONMENT}DB"}
server.base=/apps/D2CommServer
smtp.host=localhost
home.domain=${env."${params.ENVIRONMENT}_home_domain"}
">someproject.properties
				"""
				
				sh """
				echo "version: '3'
services:
  app:
    container_name: ${JOB_NAME}_latest
    image: ${docker_registry}/${JOB_NAME}:${env.branch_name}
    restart: always
    ports:
      - "${APP_EXTPORT}:8080"
    volumes:
      - /tmp/Data/${JOB_NAME}/logs:/opt/apache-tomcat/logs
      - /tmp/Data/${JOB_NAME}/webapps:/opt/apache-tomcat/webapps
">docker-compose.yaml
				"""
				
				sh """
				echo "
FROM centos:7
USER root
RUN yum install -y wget fontconfig

RUN wget http://someproject.proj.local/soft/jdk1.${JavaVersion}.tar -P /opt/
RUN wget http://someproject.proj.local/soft/apache-${TomcatVersion}.tar.gz -P /opt/
RUN tar -xf /opt/jdk1.${JavaVersion}.tar -C /opt/
RUN tar -xf /opt/apache-${TomcatVersion}.tar.gz -C /opt/
RUN rm -rf /opt/jdk1.${JavaVersion}.tar
RUN rm -rf /opt/apache-${TomcatVersion}.tar.gz

RUN wget -r -nH --cut-dirs=2 --no-parent http://someproject.proj.local/fonts/ -P /usr/share/fonts/
RUN fc-cache -fv /usr/share/fonts/
ENV JAVA_HOME /opt/jdk1.${JavaVersion}
ENV CATALINA_HOME /opt/apache-tomcat
ENV PATH /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/jdk1.${JavaVersion}/bin:/opt/apache-tomcat/bin:/opt/apache-tomcat/scripts
RUN chmod +x /opt/apache-tomcat/bin/*
ADD *.war /opt/apache-tomcat/webapps/
ADD someproject.properties /opt/apache-tomcat/conf/
WORKDIR /opt/apache-tomcat/
EXPOSE 8080
CMD /opt/apache-tomcat/bin/catalina.sh run">Dockerfile
				"""
			}
		}
	}
	stage('push to Artifactory') {
		steps {
			script{
				rtUpload (
					serverId: "${artifactory_ID}",
					spec: '''{
					"files": [
						{
						"pattern": "target/*.jar",
						"target": "${artifactory_target_folder}/"
						}
					]
					}''',
					buildName: "$JOB_NAME",
					buildNumber: "$BUILD_NUMBER")
			}
		}	
	}
	stage('Building image') {
		steps{
			script {
				sh "docker build . --network ${service_network} -t $docker_registry/$JOB_NAME:${env.branch_name}"
				sh "docker login https://$docker_registry"
				sh "docker push $docker_registry/$JOB_NAME:${env.branch_name}"
				sh "docker tag $docker_registry/$JOB_NAME:${env.branch_name} $docker_registry/$JOB_NAME:latest"
				sh "docker push $docker_registry/$JOB_NAME:latest"
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
			sh "docker rmi -f $docker_registry/$JOB_NAME:latest"
			sh "docker rmi -f $docker_registry/$JOB_NAME:${env.branch_name}"
		}
	}
}
//	post {
//		always {
//			echo 'One way or another, I have finished'
//			deleteDir()
//		}
//	}
}
