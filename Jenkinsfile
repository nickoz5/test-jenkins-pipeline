@Library('fnms-cicd-library@master') _

@NonCPS
def changedResources() {
    List<String> transFiles = []

    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            echo "${entry.commitId} by ${entry.author} on ${new Date(entry.timestamp)}: ${entry.msg}"
            def files = new ArrayList(entry.affectedFiles)
            for (int k = 0; k < files.size(); k++) {
                def file = files[k]
                if (file.path ==~ /.*resx/)
                    transFiles << file.path
            }
        }
    }

    return transFiles
}

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

pipeline
{
    agent none

    options {
        preserveStashes(buildCount: 3)  // stashes remain for 3 build retries
        disableConcurrentBuilds()       // only build 1 branch concurrently
        skipDefaultCheckout()           // dont allow jenkins to checkout to ws root on first stage
    }

    stages {
        stage('Light Build') {
            agent {
                node {
                    label 'build'
                }
            }
            steps {
                echo 'Starting checkout....'
                checkoutRepo()

                stashSomeStuff()

                script {
                    List<String> transFiles = changedResources()

                    transFiles.each {
                        echo "found match:  ${it}"
                    }

                    writeFile file: 'changedfiles.txt', text: transFiles.join('\n')
                }
            }
            post {
                always {
                    recordIssues enabledForFailure: true, ignoreFailedBuilds: false, tools: [msBuild(id: 'light-build')]
                }
            }
        }
    }
}