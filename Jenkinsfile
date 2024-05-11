pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'Git branch name')
        booleanParam(name: 'RUN_STAGES', defaultValue: true, description: 'Whether to run stages or not')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Clone Repository') {
            steps {
                script {
                    git branch: "${params.GIT_BRANCH}",
                    url: 'https://github.com/chrisdylan237/Stomer.git'
                }
            }
        }

        stage('Setup parameters') {
            steps {
                script {
                    // Set appropriate ownership for npm cache director
                    properties([
                        parameters([
                            string(name: 'WARNTIME',
                                    defaultValue: '0',
                                    description: 'Warning time (in minutes) before starting upgrade'),
                            string(
                                    defaultValue: 'develop',
                                    name: 'Please_leave_this_section_as_it_is',
                                    trim: true
                            )
                        ])
                    ])
                }
            }
        }

        stage('warning') {
            steps {
                script {
                    notifyUpgrade(currentBuild.result, "WARNING")
                    sleep(time: params.WARNTIME.toInteger(), unit: 'MINUTES')
                }
            }
        }

        stage('Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-dylan', 
                    usernameVariable: 'USERNAME', 
                    passwordVariable: 'PASSWORD')]) {
                        sh "docker login -u ${USERNAME} -p ${PASSWORD}"
                        sh "ls -a"
                        sh "pwd"
                    }
                }
            }
        }

        stage('Clean workspace') {
            steps {
               cleanWs()
           }
        }

        stage('Test golang') {
            agent {
                docker {
                    image 'golang:1.20.1'
                    args '-u 0:0'
                }
            }
            steps {
                sh 'cd $WORKSPACE/stormer-project02/src/catalog/ && go test'
            }
        }

        stage('Build SonarQube Scanner CLI Image') {
            steps {
                script {
                    sh "pwd" // Add this line for debugging
                    sh "cd ${env.WORKSPACE}/stormer-project02/images/ && docker build -t chrisdylan/sonar-stomer-cli:${BUILD_NUMBER} . "
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv("SonarScanner") {
                        docker.image("chrisdylan/sonar-stomer-cli:${BUILD_NUMBER}").inside {
                            sh "sonar-scanner"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                notifyUpgrade(currentBuild.result, "POST")
            }
        }
    }
}

def notifyUpgrade(String buildResult, String whereAt) {
    def channel = 'dev-project'
    def message
    def color

    if (buildResult == "SUCCESS") {
        switch (whereAt) {
            case 'WARNING':
                message = "Stormer: Upgrade starting in ${params.WARNTIME} minutes @ ${env.BUILD_URL} Application Stormer"
                color = "#439FE0"
                break
            case 'POST':
                message = "Stormer: Upgrade completed successfully @ ${env.BUILD_URL} Application Stormer"
                color = "good"
                break
            default:
                message = "Stormer: Starting upgrade @ ${env.BUILD_URL} Application Stormer"
                color = "good"
                break
        }
    } else {
        message = "Stormer: Upgrade was not successful. Please investigate it immediately. @ ${env.BUILD_URL} Application Stormer"
        color = "danger"
    }

    slackSend(channel: channel, color: color, message: message)
}
