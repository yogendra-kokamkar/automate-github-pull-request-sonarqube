def skipRemainingStages = false

pipeline {
    agent any
    environment {
        BRANCH_NAME = ''
    }

    stages {
        stage('Read JSON') {
            steps {
                script {
                    def payload = env.payload
                    if (payload.contains('"opened"')) {
                        def jsonSlurper = new groovy.json.JsonSlurper()
                        def jsonObject = jsonSlurper.parseText(payload)
                        env.GIT_HASH = jsonObject.pull_request.head.sha
                        env.PULL_REQUEST_NUMBER = jsonObject.pull_request.number
                    } else if (payload.contains('"closed"')) {
                        echo 'Pull request merged successfully!'
                        sendPullRequestClosedEmail()
                        currentBuild.result = 'SUCCESS'
                        skipRemainingStages = true
                    } else {
                        error("Payload is empty or null.")
                    }
                }
 
            }
        }
        stage('Checkout') {
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                script {
                    sh "git checkout ${env.GIT_HASH}"
                }
            }
        }
        stage('SonarQube analysis') {
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'sonar-scanner -Dsonar.projectKey=<SONARQUBE_PROJECT_KEY> -Dsonar.organization=<SONARQUBE_ORGANIZATION> -Dsonar.sources=./src'
                }
            }
            post {
                success {
                    echo 'SonarQube scan passed!'
                    emailext subject: 'SonarQube scan passed',
                             body: "SonarQube scan for your project has passed successfully! You can view the report [here](https://sonarcloud.io/dashboard?id=<SONARQUBE_PROJECT_KEY>)",
                             to: 'example@email.com'
                }
                failure {
                    echo 'SonarQube scan failed!'
                    emailext subject: 'SonarQube scan failed',
                             body: "SonarQube scan for your project has failed. Please check the build logs for more details. You can view the report [here](https://sonarcloud.io/dashboard?id=<SONARQUBE_PROJECT_KEY>)",
                             to: 'example@email.com'
                }
            }
        }
        stage('Manual Approval') {
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                script {
                    approved = input(
                        message: 'Proceed with the Merge?',
                        parameters: [
                            choice(name: 'ACTION', choices: 'Yes\nNo', description: 'Choose whether to proceed or not')
                        ]
                    )
                    if (approved == 'Yes') {
                        echo 'Deployment approved!'
                        acceptPullRequest()
                    } else {
                        error 'Deployment aborted!'
                    }
                }
            }
        }
    }
}

def acceptPullRequest() {
    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
        def response = sh(script: "curl -X PUT -H 'Authorization: token ${GITHUB_TOKEN}' -H 'Accept: application/vnd.github.v3+json' 'https://api.github.com/repos/<GITHUB_ACCOUNT_NAME>/<GITHUB_REPO_NAME>/pulls/${env.PULL_REQUEST_NUMBER}/merge'", returnStdout: true)
        if (response.contains('"merged": true')) {
            echo 'Pull request merged successfully!'
            //sendPullRequestClosedEmail()
        } else {
            error 'Failed to merge pull request!'
            sendPullRequestfailedEmail()
        }
    }
}

def sendPullRequestClosedEmail() {
    emailext subject: 'Pull Request Merged',
             body: 'The pull request has been Merged.',
             to: 'example@email.com'
}

def sendPullRequestfailedEmail() {
    emailext subject: 'Pull Request Merge Failed',
             body: 'The pull request Merge has been Failed.',
             to: 'example@email.com'
}
