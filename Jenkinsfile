pipeline {
    agent {
        kubernetes {
            label 'node-carbon'
        }
    }
    environment {
        npm_config_registry = 'http://nexus.molgenis-nexus:8081/repository/npm-central/'
    }
    stages {
        stage('Prepare') {
            steps {
                script {
                    env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                }
                container('vault') {
                    script {
                        env.GITHUB_TOKEN = sh(script: 'vault read -field=value secret/ops/token/github', returnStdout: true)
                        env.REGISTRY_CRED_USR = sh(script: 'vault read -field=username secret/ops/account/nexus', returnStdout: true)
                        env.REGISTRY_CRED_PSW = sh(script: 'vault read -field=password secret/ops/account/nexus', returnStdout: true)
                    }
                }
            }
        }
        stage('Build: [ pull request ]') {
            when {
                changeRequest()
            }
            steps {
                container('node') {
                    sh "yarn install"
                    sh "yarn build"
                }
            }
        }
        stage('Build: [ master ]') {
            when {
                branch 'master'
            }
            steps {
                milestone 1
                container('node') {
                    sh "yarn install"
                    sh "yarn build"
                }
            }
        }
        stage('Release: [ master ]') {
            when {
                branch 'master'
            }
            environment {
                ORG = 'molgenis'
                APP_NAME = 'molgenis-app-lifecycle-manuals'
                REGISTRY = 'registry.molgenis.org'
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    script {
                        env.RELEASE_SCOPE = input(
                                message: 'Do you want to release?',
                                ok: 'Release',
                                parameters: [
                                        choice(choices: 'patch\nminor\nmajor', description: '', name: 'RELEASE_SCOPE')
                                ]
                        )
                    }
                }
                milestone 2
                container('node') {
                    sh "git config --global user.email molgenis+ci@gmail.com"
                    sh "git config --global user.name molgenis-jenkins"
                    sh "git remote set-url origin https://${GITHUB_TOKEN}@github.com/${ORG}/${APP_NAME}.git"

                    sh "git checkout -f ${BRANCH_NAME}"

                    sh "npm config set unsafe-perm true"
                    sh "npm version ${RELEASE_SCOPE} -m '[ci skip] [npm-version] %s'"

                    sh "git push --tags origin ${BRANCH_NAME}"
                }
            }
        }
    }
    post {
        // [ slackSend ]; has to be configured on the host, it is the "Slack Notification Plugin" that has to be installed
        success {
            notifySuccess()
        }
        failure {
            notifyFailed()
        }
    }
}

def notifySuccess() {
    slackSend(channel: '#releases', color: '#00FF00', message: 'JS-module-build is successfully deployed on https://registry.npmjs.org: Job - <${env.BUILD_URL}|${env.JOB_NAME}> | #${env.BUILD_NUMBER}')
}

def notifyFailed() {
    slackSend(channel: '#releases', color: '#FF0000', message: 'JS-module-build has failed: Job - <${env.BUILD_URL}|${env.JOB_NAME}> | #${env.BUILD_NUMBER}')
}