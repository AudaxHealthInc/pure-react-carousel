@Library('global-pipeline-libraries') _

pipeline {
    agent any
    environment {
        npm_config_cache = 'npm-cache'
        ARTIFACTORY_CREDENTIALS = credentials('artifactory')
    }
    parameters {
        choice(name: 'release', choices: 'rally-versioning\npatch\nminor\nmajor', description: 'Type of release to make.  Use rally-versioning for a SNAPSHOT')
    }

    stages {
        stage('Build, Test & Release') {
            agent {
                docker {
                    image 'docker.werally.in/node:10.16.0'
                    label "ec2"
                    args "-u root:root -v \$HOME/.ivy2/cache:/root/.ivy2/cache"
                    reuseNode true
                }
            }
            stages {
                stage('Set up Github') {
                    steps {
                        script {
                            rally_git_setDefaultCredentials()
                        }
                    }
                }
                stage('Clear local tags') {
                    steps {
                        sh 'git tag -d $(git tag -l)'
                    }
                }
                stage('Checkout scm') {
                    steps {
                        rally_git_branchCheckout()
                    }
                }
                stage('Setup release variables') {
                    steps {
                        script {
                            env.previousVersion = rally_git_closestTag()
                            echo "previousVersion: ${env.previousVersion}"
                            env.newVersion = rally_git_nextTag(params.release)
                            echo "newVersion: ${env.newVersion}"
                            env.appName = "pure-react-carousel"
                            echo "appName: ${env.appName}"
                        }
                    }
                }
                stage('Artifactory credentials setup') {
                    steps {
                        sh 'npm config set registry https://artifacts.werally.in/artifactory/api/npm/npm'
                        sh 'npm config set _auth $(echo -n $ARTIFACTORY_CREDENTIALS | base64)'
                        sh 'npm config set always-auth true'
                    }
                }
                stage('Install') {
                    steps {
                        sh 'npm ci'
                    }
                }
                stage('Test') {
                    steps {
                        sh 'npm run test'
                    }
                }
                stage('Publish Snapshot') {
                    when {
                        allOf {
                            expression { params.release == 'rally-versioning' }
                        }
                    }
                    steps {
                        sh "npm version ${env.newVersion}"
                        sh "npm publish"
                        sh "echo remove snapshot tag from local git"
                        sh "git tag -d ${env.newVersion}"
                    }
                }
                stage('Publish Version') {
                    when {
                        allOf {
                            expression { params.release != 'rally-versioning' }
                        }
                    }
                    environment {
                        GITHUB_TOKEN = credentials('github-token')
                    }
                    steps {
                        withCredentials([
                            usernamePassword(credentialsId: 'github-token', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')
                        ]) {
                            script {
                                rally_git_setDefaultCredentials()
                                sh "npm run release -- ${params.release} --npm.skipChecks"
                            }
                        }
                    }
                }
            }
        }
        stage('End') {
            steps {
                script {
                    currentBuild.description = "${env.newVersion}"
                    echo "**** ${params.release.toUpperCase()} VERSION CREATED: ${env.newVersion} ****"
                }
            }
        }
    }
}
