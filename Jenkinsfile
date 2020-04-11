pipeline {
agent any
	environment {
		STAGE = 'stageserver.projects.local'
		TEST = 'testserver.projects.local'
		PROD = 'prodstageserver.projects.local'
		docker_registry = 'myproject.domain.local:5000'
		git_url = 'https://git.domain.local/test_git.git'
		git_cred_id = 'b01ac2e6-ba0d-40cb-834b-9b0a883f9600' 
		APP_EXTPORT = '30100'
		service_network = "docker_app_net"
		Maven_OPTS = '-Dmaven.test.failure.ignore'
		artifactory_user = 'aaaaeeeeeeeeeee'
		artifactory_password = 'assssssssssss'
		artifactory_url = 'http://myproject.domain.local/artifactory'
		artifactory_target_folder = 'test-repo'
	}
	parameters {
//	    booleanParam(defaultValue: true, description: '', name: 'LatestTag')
//	    booleanParam(defaultValue: false, description: '', name: 'LatestCommitToBranch')
		choice(name: 'MavenVersion', choices: "Maven 3.3.3", description: 'On this step you need select Maven Version')
		choice(name: 'JavaVersion', choices: "8", description: 'On this step you need select JAVA Version')
		choice(name: 'TomcatVersion', choices: "tomcat-8", description: 'Please select ENV server for deploy you APP')
		choice(name: 'Deploing', choices: "NO", description: 'Option for allow/decline deploy to ENV')
		choice(name: 'ENVIRONMENT', choices: "TEST", description: 'Please select ENV server for deploy you APP')
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
    container_name: ${JOB_NAME}_${env.branch_name}
    image: ${docker_registry}/${JOB_NAME}:${env.branch_name}
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
RUN yum install -y wget
WORKDIR /opt/
RUN wget http://myproject.domain.local/soft/jdk1.${JavaVersion}.tar
RUN tar -xf /opt/jdk1.${JavaVersion}.tar -C /opt/
ENV JAVA_HOME /opt/jdk1.${JavaVersion}/
RUN export JAVA_HOME
RUN rm -rf /opt/jdk1.${JavaVersion}.tar
RUN wget http://myproject.domain.local/soft/apache-${TomcatVersion}.tar.gz
RUN tar -xf /opt/apache-${TomcatVersion}.tar.gz -C /opt/
RUN rm -rf /opt/apache-${TomcatVersion}.tar.gz
RUN rm -rf /opt/apache-tomcat/webapps/*

COPY app.jar /opt/apache-tomcat/webapps/
RUN mkdir -p /usr/share/fonts/patagonia_care/ && cd /usr/share/fonts/patagonia_care/
RUN wget -r -nH --cut-dirs=2 --no-parent http://myproject.domain.local/fonts/
RUN fc-cache -fv /usr/share/fonts/patagonia_care/
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
	post {
		always {
			echo 'One way or another, I have finished'
			deleteDir()
		}
	}
}
