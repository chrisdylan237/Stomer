pipeline {
    agent any
    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Setup parameters') {
            steps {
                script {
                    // Set appropriate ownership for npm cache directory
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
                    notifyUpgrade(currentBuild.currentResult, "WARNING")
                    sleep(time: env.WARNTIME.toInteger(), unit: 'MINUTES')
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
                    }
                }
            }
        }

        stage('clean env') {
            steps {
                sh 'docker system prune -fa || true'
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

        // Add similar stages for testing other components...

        stage('SonarQube analysis') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:4.8.0'
                }
            }
            environment {
                CI = 'true'
                scannerHome = '/opt/sonar-scanner'
            }
            steps {
                withSonarQubeEnv('SonarScanner') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        // Add similar stages for building other components.       
         
    }

    post {
        always {
            script {
                notifyUpgrade(currentBuild.currentResult, "POST")
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
                message = "Stormer: Upgrade starting in ${env.WARNTIME} minutes @ ${env.BUILD_URL} Application Stormer"
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
