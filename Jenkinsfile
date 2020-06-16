def programId = 0
def pipelineId = 0
def remoteBranch = 'develop'
def gitURL = 'git@github.com:rbotha78/aem-guides-wknd.git'
def cmURL = 'git.cloudmanager.adobe.com/emeaaem/Hackathon-EMEAAEMConsultingProgram-p13954'

pipeline {
    agent any

    stages {
        stage('Git Checkout branch') {
            steps {
                checkout([
                        $class: 'GitSCM',
                        branches: [[name: env.BRANCH_NAME]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [[$class: 'CleanBeforeCheckout',
                                      deleteUntrackedNestedRepositories: true]],
                        submoduleCfg: [],
                        userRemoteConfigs: [[credentialsId: 'github',
                                             url: gitURL]]])
            }
        }

        stage('Validate configs') {
            steps {
                script {
                    def config = readJSON file: 'cloudmanager/config.json'

                    programId = config['program'];
                    pipelineId = config['pipeline-mapping'][env.BRANCH_NAME]['pipeline'];
                    remoteBranch = config['pipeline-mapping'][env.BRANCH_NAME]['remote'];

                    echo "Program: ${programId}"
                    echo "Pipeline: ${pipelineId}"
                    echo "Remote Branch: ${remoteBranch}"
                }
            }
        }

        stage('Push Code to Cloud Manager') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'cm-creds', passwordVariable: 'pass', usernameVariable: 'user')]) {
                    // the code in here can access $pass and $user
                    sh("""
                        echo "Adding CM git repo remote"
                        git remote add cm-repo "https://$user:$pass@${cmURL}"
    
                        echo "Pushing to CM repo"
                        git push -f cm-repo HEAD:refs/heads/${remoteBranch}
                        
                    """
                    )
                }
            }
        }

        stage('Start Cloud Manager Build') {
            steps {
                step([$class: 'CloudManagerBuilder', pipeline: pipelineId.toString(), program: programId.toString(), polling: true])
            }
        }

        stage('Gather Advance Parameters') {
            input {
                message "Did build pass validation?"
                parameters {
                    choice(
                            choices: ['pass', 'fail'],
                            description: 'Validation',
                            name: 'VALIDATION')
                }
            }
            when {
                expression { VALIDATION == 'fail' }
            }
            steps {
                timeout(time: 30, unit: 'SECONDS') {
                    script {
                        // Show the select input modal
                        def INPUT_PARAMS = input message: 'Override metrics', ok: 'Override',
                                parameters: [
                                        choice(name: 'METRIC_1', choices: ['Yes', 'No'].join('\n'), description: 'Override metric 1'),
                                        choice(name: 'METRIC_2', choices: ['Yes', 'No'].join('\n'), description: 'Override metric 2')]

                        env.OVERRIDE = [INPUT_PARAMS.METRIC_1, INPUT_PARAMS.METRIC_2].join('\n')
                    }
                }
            }
        }
        stage('Use Advance Parameters') {
            steps {
                script {
                    echo "Override: ${env.OVERRIDE}"
                }
            }
        }
    }

    post {
        always {
            script {
                sh """
                echo 'Running after the stages'

                echo "Remove CM repo remote reference"
                git remote rm cm-repo
                """
            }
        }
    }
}
