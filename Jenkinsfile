@Library('fnms-cicd-library@master') _

def checkoutRepo() {
    checkout([
        $class: 'GitSCM',
        branches: [[name: '*/git-pipeline']],
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
    
    def fnms_repo = params.containsKey('url') ? params.fnmsRepo : 'https://github.com/nickoz5/test-jenkins-pipeline'
    def fnms_repo_branch = "*/${BRANCH_NAME}"
    def credential_name = params.containsKey('credentials') ? params.credentials : 'eng-github-access'

    checkout([
        $class: 'GitSCM',
        branches: [[name: fnms_repo_branch]],
        doGenerateSubmoduleConfigurations: false,
        extensions:
        [
            [$class: 'CleanCheckout'],
            [$class: 'CloneOption', noTags: true, timeout: 120, shallow: true],
            [$class: 'RelativeTargetDirectory', relativeTargetDir: './src']
        ],
        submoduleCfg: [],
        userRemoteConfigs:
        [
            [
                credentialsId: credential_name,
                url: fnms_repo,
                refspec: "+refs/heads/${BRANCH_NAME}:refs/remotes/origin/${BRANCH_NAME}"
            ]
        ]
    ])
}

def stashSomeStuff() {
 
    def buildSupportFiles = '''
src/App.config,
src/Form1.cs,
src/WindowsFormsApp1.csproj,
src/obj/Debug/WindowsFormsApp1.exe
'''
     stash includes: buildSupportFiles, name: 'build-support'
}

pipeline {
    agent none
    
    options {
        preserveStashes(buildCount: 3)  // stashes remain for 3 build retries
        disableConcurrentBuilds()       // only build 1 branch concurrently
        skipDefaultCheckout()           // dont allow jenkins to checkout to ws root on first stage
    }

    stages {
        stage('Build') {
            parallel {
                stage('Light Build') {
                agent {
                    node {
                        label 'build'
                    }
                }            
                steps {
                    echo "Starting checkout...."
                    checkoutRepo()

                    powershell script: '''
                        . .\\.build-support\\support\\functions.ps1
                        .\\.build-support\\support\\environment.ps1
                        & nuget.exe restore src
                        Invoke-Build -WorkingDirectory .\\src -BuildFile 'WindowsFormsApp1.sln' -Targets @('Build')
    '''

                     powershell script: 'write-host $Env:BUILD_ID'
                    
                    stashSomeStuff()
                }
            post {
                always {
                    recordIssues enabledForFailure: true, ignoreFailedBuilds: false, tools: [msBuild(id: 'light-build')]
                }
            }
            }
             stage('Full Build') {
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
                        & nuget.exe restore src
                        Invoke-Build -WorkingDirectory .\\src -BuildFile 'WindowsFormsApp1.sln' -Targets @('Build')
    '''

                     powershell script: 'write-host $Env:BUILD_ID'

                }
                post {
                    always {
                        recordIssues enabledForFailure: true, ignoreFailedBuilds: false, tools: [msBuild(id: 'full-build')]
                    }
                }
             }
            }           
        }
        stage ('Unit Test') {
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
                        & nuget.exe restore src
                    Invoke-Build -WorkingDirectory .\\src -BuildFile 'WindowsFormsApp1.sln' -Targets @('Build')
                    mstest /testcontainer:.\\src\\UnitTestProject1\\bin\\Debug\\UnitTestProject1.dll /resultsfile:tests.trx
'''
            }
            post {
                always {
                     mstest testResultsFile: '*.trx'
                }
            }
        }
        stage('Deploy') {
            agent {
                node {
                    label 'build'
                }
            }
            when {
                tag "release-*"
            }
            steps {
                echo '${TAG_NAME}'
            }
        }
    }
            post {
               failure {

                    script {
                        def failedStages = fnms.getFailedStages( currentBuild )
                        echo "Failed stages:\n" + failedStages.join('\n')                        
                    }
                   
                  sendNotifications(currentBuild.currentResult)
               }
               fixed {
                  sendNotifications(currentBuild.currentResult)
               }
            }
}
