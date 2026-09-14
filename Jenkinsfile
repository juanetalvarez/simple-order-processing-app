pipeline {
    agent {
        label 'mac-bycontroller-agent'
    }
    tools {
        maven 'maven-3.9.9'
        jfrog 'jfrog-cli'
    }

    stages {
        stage('Checks for [skip ci] and Synchronize local tags/branches with remote') {
            steps {
                // Checks the last commit for [ci skip] or [skip ci] and deletes the triggered build
                scmSkip(deleteBuild: true, skipPattern: '.*(\\[ci skip\\]|\\[skip ci\\]).*')
                // Fetch remote tags/branches and delete local tags/branches that don't exist on the remote
                sh 'git fetch --tags --prune-tags --prune --force'
            }
        }
        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'
                // Executes test cases (e.g., JUnit)
                sh 'mvn test'
            }
            post {
                always {
                    // Optional: Captures and visualizes test reports in Jenkins UI
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('Run Integration Tests'){
            steps{
                echo 'Running Integration Tests'
            }
        }
        stage('Extract artifactId and Calculate Next Version') {
            when {
                branch 'main'
            }
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
            when {
                branch 'main'
            }
            steps {
                // Update version in pom.xml using Maven versions plugin and package
                sh '''
                    mvn versions:set -DnewVersion=${NEW_VERSION} -DgenerateBackupPoms=false
                    mvn package
                '''
            }
        }
        stage('Commit and Tag') {
            when {
                branch 'main'
            }
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
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ['github-ssh-key']) {
                    sh 'git push origin HEAD:refs/heads/main --tags'
                }
            }
        }

        stage('Publish to Artifactory') {
            when {
                branch 'main'
            }
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
            when {
                branch 'main'
            }
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

        // Manual sign-off approval
        stage('Approval for PROD') {
            when {
                branch 'main'
            }
            steps {
                script {
                    input message: "Promote ${ARTIFACT_ID}-${NEW_VERSION}.jar to PROD environment?",
                          ok: 'Yes, Proceed to PROD'
                }
            }
        }
        stage('Promote to PROD') {
            when {
                branch 'main'
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
            echo 'Pipeline Executed Successfully.'
        }
        failure {
            echo 'Build Failed. Please Check LOGS.'
        }
    }

}
