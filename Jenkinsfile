pipeline {
    agent { label "linux || macosx || windows" }
    parameters {
        string (
            defaultValue: '^(master)$',
            description: 'Regular expression of branch names for which a deploy action would be attempted after a successful build and test; leave empty to not deploy',
            name : 'DEPLOY_BRANCH_PATTERN')
        string (
            defaultValue: '',
            description: 'Name of your job that handles deployments and should accept arguments: DEPLOY_GIT_URL DEPLOY_GIT_BRANCH DEPLOY_GIT_COMMIT -- and it is up to that job what to do with this knowledge (e.g. git archive + push to packaging); leave empty to not deploy',
            name : 'DEPLOY_JOB_NAME')
        booleanParam (
            defaultValue: true,
            description: 'If the deployment is done, should THIS job wait for it to complete and include its success or failure as the build result (true), or should it schedule the job and exit quickly to free up the executor (false)',
            name: 'DEPLOY_REPORT_RESULT')
    }
    triggers {
        pollSCM 'H/5 * * * *'
    }
    stages {
        stage ('compile') {
            steps {
                dir ('src') {
                    sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make -k -j4 || make'
                    sh 'echo "Are GitIgnores good after make? (should have no output below)"; git status -s || true'
                }
            }
        }
        stage ('test') {
            steps {
                dir ('src') {
                    sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make test'
                    // The test is trivial (run the binary to see if it works and prints the version), no git status to validate
                }
            }
        }
        stage ('self-check GSL parser') {
            steps {
                dir ('src') {
                    sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make check'
                    sh 'echo "Are GitIgnores good after make check? (should have no output below)"; git status -s || true'
                }
            }
        }
        stage ('deploy if appropriate') {
            steps {
                script {
                    if ( (params["DEPLOY_JOB_NAME"] != "") && (params["DEPLOY_BRANCH_PATTERN"] != "") ) {
                        if ( env.BRANCH_NAME =~ params["DEPLOY_BRANCH_PATTERN"] ) {
                            GIT_URL = sh(returnStdout: true, script: """git remote -v | egrep '^origin' | awk '{print \$2}' | head -1""").trim()
                            GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse --verify HEAD').trim()
                            build job: "${params["DEPLOY_JOB_NAME"]}", parameters: [
                                string(name: 'DEPLOY_GIT_URL', value: "${GIT_URL}"),
                                string(name: 'DEPLOY_GIT_BRANCH', value: env.BRANCH_NAME),
                                string(name: 'DEPLOY_GIT_COMMIT', value: "${GIT_COMMIT}")
                                ], quietPeriod: 0, wait: params.DEPLOY_REPORT_RESULT, propagate: params.DEPLOY_REPORT_RESULT
                        } else {
                            echo "Not deploying because branch '${env.BRANCH_NAME}' did not match filter '${params.DEPLOY_BRANCH_PATTERN}'"
                        }
                    } else {
                        echo "Not deploying because deploy-job parameters are not set"
                    }
                }
            }
        }
    }
}
