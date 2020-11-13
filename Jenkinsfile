def checkoutRepo() {
    checkout([
        $class: 'GitSCM',
        branches: [[name: '*/master']],
        doGenerateSubmoduleConfigurations: false,
        changelog: false, poll: false,
        extensions:
        [
            [$class: 'CloneOption', timeout: 120],
            [
                $class: 'RelativeTargetDirectory',
                relativeTargetDir: './.build-support'
            ]
        ],
        submoduleCfg: [],
        userRemoteConfigs:
        [
            [
                credentialsId: 'eng-github-access',
                url: 'https://github.com/flexera/fnms-build'
            ]
        ]
    ])
}

pipeline {
    agent none
    stages {
        stage('Build') {
            agent {
                node {
                    label 'build'
                }
            }            
            steps {
                checkoutRepo()

                powershell script: '''
                    $ErrorActionPreference = 'Stop';
                    . .\\.build-support\\support\\functions.ps1
                    .\\.build-support\\support\\environment.ps1
                    Invoke-Build -WorkingDirectory . -BuildFile 'WindowsFormsApp1.sln' -Targets @('Build')
'''
            }
        }
    }
}
