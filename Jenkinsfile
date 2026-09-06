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
        stage('Parallel Build and Test') {
            parallel {
                stage('Build') {
                    steps{
                        sh 'mvn clean compile'
                    }
                }
                stage('Unit Tests') {
                    steps{
                        sh 'mvn test'
                    }
                }
            }
        }
        stage('Package') {
            steps{
                sh 'mvn package'
            }
        }
        stage('Run Integration Tests'){
            when {
                expression { params.DEPLOY_ENV != 'prod' }
            }
            steps{
                echo 'Running Integration Tests'
            }
        }

        stage('Approval') {
            steps{
                input message: "Do you want to procced to deployment?"
                ok: 'Yes, Deploy'
                submitter: 'admin'
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
