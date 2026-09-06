pipeline {
    agent any
    stages {
        stage('Checkout Code') {
            steps{
                git branch: 'main', url: 'https://github.com/juanetalvarez/simple-order-processing-app.git'
            }
        }

        stage('Build with Maven') {
            steps{
                sh '/opt/maven/apache-maven-3.9.9/bin/mvn clean package'
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
