pipeline {
    agent {
        label 'mac-bycontroller-agent'
    }
    tools {
        maven 'maven-3.9.9'
    }
    parameters {
        string(name: 'DEPLOY_ENV',
            defaultValue: 'dev',
            description: 'Environment to Deploy (dev or prod)'
        )
    }
    stages {
        stage('Checkout Code') {
            steps{
                git branch: 'main', url: 'https://github.com/juanetalvarez/simple-order-processing-app.git'
            }
        }

        stage('Build with Maven') {
            steps{
                sh 'mvn clean package'
            }
        }

        stage('Deploy to PROD') {
            when {
                expression { params.DEPLOY_ENV == 'prod' }
            }
            steps {
                echo 'Deploying Application to PRODUCTION...'
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
