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
          if [ -f target/myapp.war ]; then
            sudo cp target/myapp.war ${TOMCAT_WEBAPPS}/
            echo 'Deployed myapp.war to tomcat webapps'
          else
            echo 'WAR not found!'
            exit 1
          fi
        """
      }
    }
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
