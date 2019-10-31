@Library("global-pipeline-libraries") _

// if we want to use specific docker image, then uncomment following line
def nodeDockerImage = 'docker.werally.in/node:10.15.3-stretch'
def shouldBuild = true

pipeline {
    agent {
        docker {
            image nodeDockerImage // we can specify this if we truly want a specific image
            label 'ec2'
        }
    }
    environment {
        // We set home to the current directory because HOME is used by npm to figure out where to put
        // the npm cache. By default, it's set to the root of the filesystem, which the Jenkins user can't
        // access. This means that files can't be written to the cache, and npm ci fails with permisison denied issues.
        HOME='.'
        ARTIFACTORY_CREDENTIALS = credentials('artifactory')
    }

    parameters {
      booleanParam(defaultValue: true, description: 'Execute pipeline?', name: 'shouldBuild')
    }

    stages {

        stage ("Should we build?") {
            steps {
                script {
                    result = sh (script: "git log -1 | grep '.*\\[ci-skip\\].*'", returnStatus: true)
                    if (result == 0) {
                        echo ("'ci-skip' spotted in git commit. Aborting.")
                        shouldBuild = false
                    }
                }
                echo 'shouldBuild value is: '
                echo shouldBuild.toString()
            }
        }

        // Best to have some sort of unit test, therefore good to keep
        stage('Test') {
            steps {
                sh 'npm ci'
                sh 'npm test'
            }
        }

        stage('Publish to Artifactory') {
            when {
                allOf {
                    branch 'master'
                    expression {
                        return shouldBuild != false
                    }
                }
            }

            steps {
                checkout scm: [
                    $class: 'GitSCM',
                    branches: scm.branches,
                    extensions: [[$class: 'CloneOption', noTags: false, reference: '', shallow: false]],
                    userRemoteConfigs: scm.userRemoteConfigs
                ]
                rally_git_setDefaultCredentials()
                echo 'Publishing to Artifactory...'
                sh """
                    curl -u${env.ARTIFACTORY_CREDENTIALS} https://artifacts.werally.in/artifactory/api/npm/auth > .npmrc
                    npm config set registry https://artifacts.werally.in/artifactory/api/npm/npm
                """
                sh '''
                  git config user.email "jenkins@rallyhealth.com"
                  git config user.name "jenkins"
                  git config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
                  git fetch --all
                  git checkout master
                  git pull
                '''
                sh 'npm version patch -m "Bumped to version: %s [ci-skip]"'
                sh 'npm publish'
                sh "git push"
            }
        }
    }

    post {
        always {
            script {
                deleteDir()
            }
        }
    }
}