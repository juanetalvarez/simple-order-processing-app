pipeline {
    agent {
        label 'mac-bycontroller-agent'
    }
    tools {
        maven 'maven-3.9.9'
        jfrog 'jfrog-cli'
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
            steps{
                echo 'Running Integration Tests'
            }
        }
        stage('Publish to Artifactory') {
            steps {
                // Show the installed version of JFrog CLI.
                jf '-v'
                // Ping Artifactory.
                jf 'rt ping'
                // Upload artifac to a repository in Artifactory
                jf 'rt u **/target/*.jar simple-order-processing-app-maven-dev-local/'
                // Publish the build-info to Artifactory.
                jf 'rt bp'
            }
        }
        stage('Deploy to DEV') {
            steps {
                echo 'Starting Application Deployment to DEV...'
                sh '''
                    echo "Creating deployment directory..."
                    mkdir -p /tmp/dev/simple-order-processing-app-deploy/

                    echo "Copying JAR to deployment directory"
                    cp target/*.jar /tmp/dev/simple-order-processing-app-deploy/

                    echo "Listing deployed files"
                    ls -l /tmp/dev/simple-order-processing-app-deploy/
                '''
            }
        }

        stage('Approval') {
            steps{
                input message: "Do you want to procced to deployment PROD?",
                ok: 'Yes, Deploy to PROD',
                submitter: 'admin'
            }
        }
        stage('Deploy to PROD') {
            steps {
                echo 'Starting Application Deployment to PROD...'
                sh '''
                    echo "Creating deployment directory..."
                    mkdir -p /tmp/prod/simple-order-processing-app-deploy/

                    echo "Copying JAR to deployment directory"
                    cp target/*.jar /tmp/prod/simple-order-processing-app-deploy/

                    echo "Listing deployed files"
                    ls -l /tmp/prod/simple-order-processing-app-deploy/
                '''
            }
        }
    }
    post {
        success {
            echo 'Pipeline Executed Successfully. Application DEPLOYED.'
        }
        failure {
            echo 'Build Failed. Please Check LOGS.'
        }
    }

}
