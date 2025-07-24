Pipeline code End to end deployment:

pipeline {
    agent {
        node{
            label 'dev'
        }
    }
    tools{
        maven 'mymaven'
        nodejs 'mynode16'
    }
    environment{
        SCANNER_HOME= tool 'mysonar'
    }

    stages {
        stage('Cleanworkspace') {
            steps {
                  cleanWs();
            }
        }
        stage("code"){
            steps{
                 git 'https://github.com/venky0199/dockerwebapp.git'
            }
        }
        stage("CQA"){
            steps{
                withSonarQubeEnv("mysonar")
                {
                    sh "mvn clean verify sonar:sonar -Dsonar.projectKey=myproject"
                }
            }
        }
        stage("Quality-gates"){
            steps{
                script{
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar'
                }
            }
        }
        stage("build"){
            steps{
                 sh "mvn clean package -DskipTests"
                 sh "cp -r target Docker-app"
            }
        }
        stage("Artifact-uploader"){
            steps{
                 nexusArtifactUploader artifacts: [[artifactId: 'vprofile', classifier: '', file: 'target/vprofile-v2.war', type: 'war']], credentialsId: 'Nexus', groupId: 'com.visualpathit', nexusUrl: '54.166.82.235:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'myrepo', version: 'v2'
            }
        }
        stage("build-image"){
            steps{
                sh 'docker build -t venky9901/deploy:mynewdbimage Docker-db'
                sh 'docker build -t venky9901/deploy:mynewappimage Docker-app'
            }
        }
        stage("trivy"){
            steps{
                sh 'trivy image venky9901/deploy:mynewdbimage'
                sh 'trivy image venky9901/deploy:mynewappimage'
            }
        }
        stage("docker hub"){
            steps{
                script{
                
                         withDockerRegistry(credentialsId: 'dockerhub') {
                                      sh' docker push venky9901/deploy:mynewdbimage'
                                      sh 'docker push venky9901/deploy:mynewappimage'
                
                      }
                    }
                  }
                }
            stage("docker slack deploy"){
                steps{
                    sh 'docker stack deploy myapp --compose-file=compose.yml'
                }
            }
        }
        post{
            always{
                   slackSend channel: 'mynewchannel24', message:"${currentBuild.result}  BUILD_NUMBER: ${env.BUILD_NUMBER}  JOB_NAME: ${env.JOB_NAME}  more info at ${env.BUILD_URL}"
            }
        }
    }


    Deploying 3 tier aplication using docker swarm by same docker network:
docker create network --driver overlay networkname
 
 
 
**docker Swarm :
DB: docker service create --name devopsdb --publish 3306:3306 --mount type=volume,source=myvolume,target=/var/lin/mysql --network overlaynetworkname mydbimage
 
app: docker service create --name myapp --publish 8080:8080 --network overlaynetwork myappimage**


Docker compose:
3 tier application deployment  (network should match)
 
docker create network --driver overlay networkname
 
 
 
docker Swarm :
DB: docker service create --name devopsdb --publish 3306:3306 --mount type=volume,source=myvolume,target=/var/lin/mysql --network overlaynetworkname mydbimage
 
app: docker service create --name myapp --publish 8080:8080 --network overlaynetwork myappimage
 
 
 
docker compose:
 
---

version: '3'

services:

    devopsdb:

      container_name: devopsdb

      deploy:

        replicas: 1

      image: venky9901/deploy:mysqldbimage

      ports:

        - "3306:3306"

      volumes:

        - mydbvolume:/var/lib/mysql

      networks:

        - myoverlaynetwork
 
 
    myapp:

      container_name: myapp

      deploy:

        replicas: 1

      image: venky9901/deploy:javaappimage

      ports:

        - "8080:8080"

      volumes:

        - myappvolume:/app
 
      networks:

        - myoverlaynetwork
 
volumes:

  mydbvolume:

  myappvolume:
 
networks:

  myoverlaynetwork:

    driver: overlay
 

