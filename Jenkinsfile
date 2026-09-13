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
                git branch: 'main', credentialsId: 'github-ssh-key', url: 'git@github.com:juanetalvarez/simple-order-processing-app.git'
                // Download the latest changes from remote repository (origin) while simultaneously cleaning up deleted remote branches and local tag
                sh 'git fetch --prune --prune-tags --tags origin'
                // Checks the last commit for [ci skip] or [skip ci] and deletes the triggered build
                scmSkip(deleteBuild: true, skipPattern: '.*(\\[ci skip\\]|\\[skip ci\\]).*')
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
        stage('Extract artifactId and Calculate Next Version') {
            steps {
                script {
                    // Extract the artifactId
                    def pom = readMavenPom file: 'pom.xml'
                    env.ARTIFACT_ID = pom.artifactId
                    // Calculate next semantic version using conventional commits
                    def nextVersion = getNextSemanticVersion(
                        majorPattern: '^([Bb]reaking|[Mm]ajor).*',
                        minorPattern: '^([Ff]eat|[Ff]eature|[Rr]efactor).*',
                        patchPattern: '^([Ff]ix|[Pp]atch).*'
                    )

                    env.NEW_VERSION = nextVersion.toString()
                    echo "Calculated Next Version: ${env.NEW_VERSION}"
                }
            }
        }
        stage('Update pom.xml and package') {
            steps {
                // Update version in pom.xml using Maven versions plugin and package
                sh '''
                    mvn versions:set -DnewVersion=${NEW_VERSION} -DgenerateBackupPoms=false
                    mvn package
                '''
            }
        }
        stage('Commit and Tag') {
            steps {
                script {
                    sh '''
                        git add pom.xml
                        git commit -m "chore: release ${NEW_VERSION} [skip ci]"
                        git tag -a "${NEW_VERSION}" -m "Release ${NEW_VERSION}"
                    '''
                }
            }
        }
        stage('Publish to Git') {
            steps {
                sshagent(credentials: ['github-ssh-key']) {
                    sh ' git push origin HEAD --tags'
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
                jf 'rt u "target/*.jar" "maven-dev-local/" --flat'
                // Publish the build-info to Artifactory.
                jf 'rt bp'
            }
        }

        stage('Promote to DEV') {
            steps {
                echo 'Starting Application Deployment to DEV...'
                sh '''
                    echo "Creating deployment directory..."
                    mkdir -p /tmp/dev/simple-order-processing-app-deploy/

                    echo "Downloading JAR to deployment directory"
                '''
                // Download the artifact
                jf 'rt dl maven-dev-local/${ARTIFACT_ID}-${NEW_VERSION}.jar /tmp/dev/simple-order-processing-app-deploy/'
                sh '''
                    echo "Listing deployed files"
                    ls -l /tmp/dev/simple-order-processing-app-deploy/
                '''
            }
        }

        stage('Promote to PROD') {
            // Manual sign-off approval
            input {
                message 'Promote ${env.ARTIFACT_ID}-${env.NEW_VERSION}.jar to PROD environment?'
                ok 'Yes, Proceed to PROD'
            }
            steps {
                echo 'Starting Application Deployment to PROD...'
                sh '''
                    echo "Creating deployment directory..."
                    mkdir -p /tmp/prod/simple-order-processing-app-deploy/

                    echo "Downloading JAR to deployment directory"
                '''
                // Download the artifact
                jf 'rt dl maven-dev-local/${ARTIFACT_ID}-${NEW_VERSION}.jar /tmp/prod/simple-order-processing-app-deploy/'
                // Promote the artifact
                jf 'rt bpr ${JOB_NAME} ${BUILD_NUMBER} maven-prod-local --comment="Released via Jenkins" --copy=true'
                sh '''
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
