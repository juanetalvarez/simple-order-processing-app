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
            steps {
                // Checks the last commit for [ci skip] and deletes the triggered build
                scmSkip(deleteBuild: true, skipPattern: '.*\\[skip ci\\].*')
                git branch: 'main', url: 'https://github.com/juanetalvarez/simple-order-processing-app.git'
            }
        }
        stage('Parallel Build and Test') {
            parallel {
                stage('Build') {
                    steps {
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
        stage('Run Integration Tests'){
            steps{
                echo 'Running Integration Tests'
            }
        }
        stage('Calculate Next Version') {
            steps {
                script {
                    // Calculate next semantic version using conventional commits
                    def nextVersion = getNextSemanticVersion(
                        majorPattern: '^[Bb]reaking.*',
                        minorPattern: '^[Ff]eature.*',
                        patchPattern: '^[Ff]ix.*'
                    )

                    env.NEW_VERSION = nextVersion.toString()
                    echo "Calculated Next Version: ${env.NEW_VERSION}"
                }
            }
        }
        stage('Update pom.xml and package') {
            steps {
                // Update version in pom.xml using Maven versions plugin
                sh "mvn versions:set -DnewVersion=${NEW_VERSION} -DgenerateBackupPoms=false clean package"
            }
        }
        stage('Commit and Tag') {
            steps {
                script {
                    sh """
                        git config --local user.email "juanet.alvarez@gmail.com"
                        git config --local user.name "Juan"
                        git add pom.xml
                        git commit -m "chore: release ${NEW_VERSION} [skip ci]"
                        git fetch --prune --prune-tags origin
                        git tag -a "${NEW_VERSION}" -m "Release ${NEW_VERSION}"
                    """
                }
            }
        }
        stage('Publish to Git') {
            steps {
                withCredentials([string(credentialsId: 'github-creds', variable: 'GIT_TOKEN')]) {
                    sh "git push https://juanetalvarez:${GIT_TOKEN}@://github.com HEAD --tags"
                }
            }
        }

        stage('Publish to Artifactory') {
            steps {
                // Show the installed version of JFrog CLI.
                jf '-v'
                // Ping Artifactory.
                jf 'rt ping'
                // Upload artifac to a repository in Artifactory
                jf 'rt u **/target/*.jar maven-dev-local/'
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
                    cd /tmp/dev/simple-order-processing-app-deploy/
                '''
                // Download the artifact
                jf 'rt dl maven-dev-local/simple-order-processing-app-1.0-SNAPSHOT.jar'
                sh '''
                    cp target/*.jar /tmp/dev/simple-order-processing-app-deploy/

                    echo "Listing deployed files"
                    ls -l /tmp/dev/simple-order-processing-app-deploy/
                '''
            }
        }

        stage('Approval') {
            steps {
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
