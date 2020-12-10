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
    ])}

def stashSomeStuff() {
 
    def buildSupportFiles = '''
.build-support/support/**,
src/msbuild/**,
src/Flexera.Bootstrap.proj,
src/Flexera.Directories.props,
src/Flexera.props,
src/Flexera.Version.props,
src/FNMP.proj,
src/FNMP.VersionRoll.proj,
src/mgs/Compliance.config,
src/mgs/Compliance.nunit,
src/mgs/Ecm.config,
src/mgs/Ecm.nunit,
src/mgs/Flexera.FNMP.DatabaseTests.proj,
src/mgs/Flexera.FNMP.Directories.props,
src/mgs/Flexera.FNMP.Msggen.proj,
src/mgs/Flexera.FNMP.proj,
src/mgs/Flexera.FNMP.Tests.proj,
src/mgs/Flexera.FNMP.Version.proj,
src/mgs/Flexera.FNMP.Version.props,
src/mgs/Flexera.IM.Tests.proj,
src/mgs/imintegration.runsettings,
src/mgs/mgsbitests.runsettings,
src/mgs/nativeunittests.runsettings,
src/mgs/runtests.testrunconfig,
src/mgs/UnitTests.config,
src/mgs/UnitTests.nunit,
src/mgs/unittests.runsettings
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
            agent {
                node {
                    label 'build'
                }
            }            
            steps {
                echo "Starting checkout...."
                checkoutRepo()

                powershell script: '''
                    $ErrorActionPreference = 'Stop';
                    . .\\.build-support\\support\\functions.ps1
                    .\\.build-support\\support\\environment.ps1
                    Invoke-Build -WorkingDirectory .\\src -BuildFile 'WindowsFormsApp1.sln' -Targets @('Build')
'''
                
                stashSomeStuff()
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
}
