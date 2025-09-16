pipeline {
    agent any

    options {
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        timeout(time: 90, unit: 'MINUTES')
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'stage', 'prod'], description: 'Target environment')
        booleanParam(name: 'APPLY_CHANGES', defaultValue: true, description: 'Run Ansible in apply (not check) mode')
        booleanParam(name: 'RUN_GENERATE_CONFIGS', defaultValue: true, description: 'Run config generation step')
        booleanParam(name: 'RUN_GENERATE_INVENTORY', defaultValue: true, description: 'Regenerate Ansible inventory (if script exists)')
        booleanParam(name: 'RUN_PLAYBOOKS', defaultValue: true, description: 'Execute Ansible playbooks')
        string(name: 'DOWNSTREAM_JOBS', defaultValue: '', description: 'Comma-separated Jenkins jobs to trigger after success')
        string(name: 'EXTRA_VARS', defaultValue: '', description: 'Additional Ansible extra vars (key1=val1 key2=val2)')
        string(name: 'ANSIBLE_LIMIT', defaultValue: '', description: 'Optional --limit hosts or groups')
        string(name: 'VAULT_CREDENTIALS_ID', defaultValue: '', description: 'Jenkins Secret Text ID containing Ansible Vault password (optional)')
    }

    environment {
        PYTHONUNBUFFERED = '1'
        WORKSPACE_DIR = env.WORKSPACE
        VENV_DIR = "${env.WORKSPACE}/.venv"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Generate Configs') {
            when { expression { return params.RUN_GENERATE_CONFIGS }}
            steps {
                script {
                    if (fileExists('playbooks/01.proxmox_k8s_generate_configs.yml')) {
                        withCredentialsWrapper {
                            sh 'ansible-playbook playbooks/01.proxmox_k8s_generate_configs.yml'
                        }
                    } else {
                        echo 'Config generation playbook not found; skipping.'
                    }
                }
            }
        }

        stage('Generate Inventory') {
            when { expression { return params.RUN_GENERATE_INVENTORY }}
            steps {
                script {
                    if (fileExists('scripts/generate_inventory.sh')) {
                        sh 'chmod +x scripts/generate_inventory.sh || true'
                        withCredentialsWrapper {
                            sh 'scripts/generate_inventory.sh ${ENVIRONMENT}'
                        }
                    } else {
                        echo 'scripts/generate_inventory.sh not found; skipping.'
                    }
                }
            }
        }

        stage('Run Playbooks') {
            when { expression { return params.RUN_PLAYBOOKS }}
            environment {
                ANSIBLE_STDOUT_CALLBACK = 'yaml'
            }
            steps {
                sh 'chmod +x scripts/run_playbooks.sh || true'
                script {
                    def applyFlag = params.APPLY_CHANGES ? '' : '--check'
                    def limitFlag = params.ANSIBLE_LIMIT?.trim() ? "--limit '${params.ANSIBLE_LIMIT.trim()}'" : ''
                    def extraVars = params.EXTRA_VARS?.trim() ? "-e \"${params.EXTRA_VARS.trim()}\"" : ''
                    def yesFlag = '--yes'
                    withCredentialsWrapper {
                        sh "scripts/run_playbooks.sh ${yesFlag} ${applyFlag} ${limitFlag} ${extraVars}"
                    }
                }
            }
        }

        stage('Trigger Downstream Jobs') {
            when { expression { return params.DOWNSTREAM_JOBS?.trim() }}
            steps {
                script {
                    def jobs = params.DOWNSTREAM_JOBS.split(',').collect { it.trim() }.findAll { it }
                    for (def jobName in jobs) {
                        echo "Triggering downstream job: ${jobName}"
                        build job: jobName, wait: false, propagate: false, parameters: [
                            string(name: 'UPSTREAM_ENVIRONMENT', value: params.ENVIRONMENT),
                            booleanParam(name: 'APPLY_CHANGES', value: params.APPLY_CHANGES)
                        ]
                    }
                }
            }
        }
    }

    // Helper to wrap steps with optional Vault credential injection
    // If VAULT_CREDENTIALS_ID is set, expose VAULT_PASSWORD env var for steps
    def withCredentialsWrapper(Closure body) {
        if (params.VAULT_CREDENTIALS_ID?.trim()) {
            withCredentials([string(credentialsId: params.VAULT_CREDENTIALS_ID.trim(), variable: 'VAULT_PASSWORD')]) {
                body()
            }
        } else {
            body()
        }
    }

    post {
        always {
            archiveArtifacts allowEmptyArchive: true, artifacts: 'logs/**'
        }
        success {
            echo 'Pipeline completed successfully.'
        }
        unsuccessful {
            echo 'Pipeline did not complete successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
        aborted {
            echo 'Pipeline aborted.'
        }
    }
}
