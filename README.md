- install vscode
- install vscode docker extensions
- install docker
- install jenkins

    ```
    docker image pull jenkins/jenkins:jdk17
    ```

    ```
    docker volume create jenkinsvol1  
    ```

    ```
    docker container run -d -p 8082:8080 \ 
        -v jenkinsvol1:/var/jenkins_home \
        --name jenkins-local \
        jenkins/jenkins:jdk17
    ```

    wip
    ```
    docker container run -d \
        -u root \
        -p 8082:8080 \
        -v jenkinsvol1:/var/jenkins_home \
        -v $(which docker):/usr/bin/docker \
        -v /var/run/docker.sock:/var/run/docker.sock \
        --name jenkins-local3 \
        jenkins/jenkins:jdk17
    ```

    ```
    docker container exec \                
        jenkins-local \
        sh -c "cat /var/jenkins_home/secrets/initialAdminPassword"
    ```
  - copy to clipboard
  - right click on jenkins/jenkins:jdk17 jenkins-local -> Open in Browser
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
- create pipeline
  - Dashboard
  - Element anlegen
  - Pipeline
    ```groovy
    pipeline {
    agent 'any'
    tools {
            maven 'MAVEN_HOME'
        }
    stages {
        stage('Checkout') {
        steps {
            script {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: 'https://github.com/Feichtmeier/calctests.git']]])
            }
        }
        }
        stage('Clean') {
        steps {
            sh(script: 'mvn -e clean')
        }
        }
        stage('Test') {
        steps {
            sh(script: 'mvn test')
        }
        }
        stage('Install') {
        steps {
            sh(script: 'mvn install')
        }
        }
        stage('Package') {
        steps {
            sh(script: 'mvn compile assembly:single')
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

    ```
    pipeline {
    agent 'any'
    tools {
        maven 'MAVEN_HOME'
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
                sh(script: 'mvn -e clean')
            }
        }
        stage('Test') {
            steps {
                sh(script: 'mvn test')
            }
        }
        //stage('Package') {
        //    steps {
        //        sh(script: 'mvn install -Pproduction')
        //    }
        //}
        stage('Initialize'){
            steps {
                script {
                    def dockerHome = tool 'docker'
                    env.PATH = "${dockerHome}/bin:${env.PATH}"
                }
            }
        }
         stage('Build Image'){
            steps {
                script {
                    docker.build("my-todo")
                }
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