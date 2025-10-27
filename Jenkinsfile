pipeline {
  agent any

  tools {
    maven 'Maven3'
    jdk 'Java19'
  }

  environment {
    WAR_NAME = 'myapp.war'
    TOMCAT_WEBAPPS = '/opt/tomcat/webapps'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/priyankahotkar/myapp-tomcat-jenkins.git'
      }
    }

    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }

    stage('Deploy') {
  steps {
    sh """
      curl -T target/myapp.war "http://admin:admin123@tomcat:8080/manager/text/deploy?path=/myapp&update=true"
    """
  }
}


    // stage('Deploy to Tomcat') {
    //     steps {
    //         sh 'cp target/*.war /opt/apache-tomcat-9.0.85/webapps/'
    //     }
    // }
  }

  post {
    success {
      echo 'Deployment successful'
    }
    failure {
      echo 'Deployment failed'
    }
  }
}
