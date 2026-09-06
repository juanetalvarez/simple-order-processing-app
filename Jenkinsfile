pipeline {
    agent any
    environment {
        MAVEN_VERSION = "3.9.9"
        MAVEN_HOME="/opt/apache-maven-${env.MAVEN_VERSION}"
    }
    stages {
        stage('Checkout Code') {
            steps{
                git branch: 'main', url: 'https://github.com/juanetalvarez/simple-order-processing-app.git'
            }
        }

        stage('Build with Maven') {
            steps{
                sh '${MAVEN_HOME}/bin/mvn clean package'
            }
        }
    }
    post {
        success {
            junit '**/target/surefire-reports/TEST-*.xml'
            archiveArtifacts 'target/*.jar'
        }
        failure {
            echo 'Build Failed'
        }
    }

}
