- install vscode
- install vscode docker extensions
- install docker
- install jenkins: [Instructions](https://www.jenkins.io/doc/book/installing/docker/)

    ```
    docker container exec \                
        jenkins-blueocean \
        sh -c "cat /var/jenkins_home/secrets/initialAdminPassword"
    ```
  - copy to clipboard
  - right click on jenkins/jenkins:jdk17 jenkins-blueocean -> Open in Browser
  - create new admin with copied password, repeat password
- Install and configure maven to work with java 17
  - Jenkins verwalten/configure Jenkins
  - Plugins
  - Available Plugins
  - Search for Maven
  - Install "Maven Integration Plugin"
  - Wait until done and select "Restart Jenkins"
  - Docker does not start the container again
  - VsCode docker extension -> right click -> start -> open in browser
  - Jenkins verwalten/configure Jenkins
  - Tools
  - Scroll down
  - Maven
  - Maven hinzufügen
  - Name: MAVEN_HOME
  - Automatisch installieren
  - Version 3.8.1
  - Save
- create docker hub account and safe credentials in jenkins (global), id: dockerhub
- create pipeline
  - Dashboard
  - Element anlegen
  - Pipeline

    ```groovy
    pipeline {
        environment {
            registry = 'feichtmeier/hub'
            registryCredential = 'dockerhub'
        }
        agent 'any'
        tools {
            maven 'maven'
            dockerTool 'docker'
        }
        stages {
            stage('Checkout') {
                steps {
                    script {
                        checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: 'https://github.com/Feichtmeier/my-todo.git']]])
                    }
                }
            }
            stage('Clean') {
                steps {
                    sh(script: 'mvn clean')
                }
            }
            stage('Test') {
                steps {
                    sh(script: 'mvn test')
                }
            }
            stage('Package') {
            steps {
                    sh(script: 'mvn install -Pproduction')
            }
            }
            stage('Build') {
                steps {
                    script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                    }
                }
            }
            stage('Deploy') {
                steps {
                    script {
                        docker.withRegistry( '', registryCredential) {
                            dockerImage.push()
                        }
                    }
                }
            }
            stage('Remove Unused docker image') {
            steps{
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
            }
        }
        post {
            always {
                junit(testResults: 'target/surefire-reports/*.xml', allowEmptyResults : true)
            }
        }
    }
    ```