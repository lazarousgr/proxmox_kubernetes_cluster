pipeline {
    agent any
    
    parameters {
        booleanParam(
            name: 'FULL_CLUSTER',
            defaultValue: true,
            description: 'Deploy full cluster (master + workers) or single node only'
        )
        credentials(
            name: 'VAULT_FILE_CREDENTIAL',
            credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.FileCredentialsImpl',
            description: 'Vault configuration file',
            required: true
        )
        password(
            name: 'PROXMOX_PASSWORD',
            description: 'Proxmox root password for initial SSH key installation',
            defaultValue: ''
        )
    }
    
    environment {
        WORKSPACE_DIR = "${WORKSPACE}"
        K8S_INVENTORY = "${WORKSPACE_DIR}/inventory/k8s_vms.ini"
        PROXMOX_INVENTORY = "${WORKSPACE_DIR}/inventory/proxmox.ini"
        VAULT_FILE = "${WORKSPACE_DIR}/group_vars/vault.yml"
    }
    
    stages {
        stage('🔍 Pre-flight Checks') {
            steps {
                script {
                    echo "🚀 Starting Kubernetes Cluster Deployment"
                    echo "📋 Configuration:"
                    echo "   • Full Cluster Mode: ${params.FULL_CLUSTER}"
                    
                    // Validate vault file credential
                    if (!params.VAULT_FILE_CREDENTIAL) {
                        error "❌ VAULT_FILE_CREDENTIAL parameter is required"
                    }
                    
                    // Check workspace structure
                    sh '''
                        if [ ! -d "${WORKSPACE_DIR}/playbooks" ]; then
                            echo "❌ Playbooks directory not found"
                            exit 1
                        fi
                        echo "✅ Workspace structure validated"
                    '''
                }
            }
        }
        
        stage('🔐 Extract Vault Configuration') {
            steps {
                withCredentials([file(credentialsId: params.VAULT_FILE_CREDENTIAL, variable: 'VAULT_FILE_PATH')]) {
                    sh '''
                        cd ${WORKSPACE_DIR}
                        mkdir -p group_vars
                        cp ${VAULT_FILE_PATH} ${VAULT_FILE}
                        chmod 600 ${VAULT_FILE}
                        echo "✅ Vault configuration extracted"
                    '''
                }
            }
        }

        stage('🔐 Generate SSH Keys') {
            steps {
                sh """
                    cd ${WORKSPACE_DIR}
                    export SSH_KEY_DIR=${WORKSPACE_DIR}/.ssh
                    ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys_merge.yml
                """
            }
        }

        stage('🔑 Install SSH Keys on Proxmox') {
            steps {
                script {
                    if (!params.PROXMOX_PASSWORD) {
                        error "❌ PROXMOX_PASSWORD parameter is required for initial SSH key installation"
                    }
                }
                sh """
                    cd ${WORKSPACE_DIR}
                    
                    # Generate configurations with password authentication (needed for inventory)
                    echo "📋 Generating inventory and configurations with password authentication..."
                    ansible-playbook  \
                        playbooks/01a.proxmox_k8s_generate_configs_with_password.yml \
                        -e "include_workers=${params.FULL_CLUSTER}"
                    
                    # Install SSH key using Ansible
                    echo "🔑 Installing public key on Proxmox using Ansible..."
                    ansible-playbook  \
                        -i ${PROXMOX_INVENTORY} \
                        playbooks/00a.proxmox_install_ssh_key.yml
                    
                    # Regenerate configurations with SSH key authentication
                    echo "🔄 Regenerating configurations with SSH key authentication..."
                    ansible-playbook  \
                        playbooks/01b.proxmox_k8s_generate_configs_with_key.yml \
                        -e "include_workers=${params.FULL_CLUSTER}"
                """
            }
        }
        
        stage('🏗️ Infrastructure Setup') {
            steps {
                sh """
                    cd ${WORKSPACE_DIR}
                    
                    # Create cloud template
                    echo "☁️ Creating VM template..."
                    ansible-playbook  \
                        -i ${PROXMOX_INVENTORY} \
                        playbooks/02.proxmox_k8s_create_vm_template.yml
                    
                    # Clone VMs
                    echo "🖥️ Cloning VMs..."
                    ansible-playbook  \
                        -i ${PROXMOX_INVENTORY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/03.proxmox_k8s_clone_vms.yml
                    
                    # Start VMs
                    echo "🚀 Starting VMs..."
                    ansible-playbook  \
                        -i ${PROXMOX_INVENTORY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/04.proxmox_k8s_start_vms.yml
                """
            }
        }
        
        stage('⚙️ System Preparation') {
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    
                    # System preparation (swap, sysctl, modules)
                    ansible-playbook  \
                        -i ${K8S_INVENTORY} \
                        playbooks/06.proxmox_k8s_os_prep.yml
                    
                    # Configure hostnames
                    ansible-playbook  \
                        -i ${K8S_INVENTORY} \
                        playbooks/05.proxmox_k8s_vms_hostname.yml
                    
                    # Install Docker/containerd
                    ansible-playbook  \
                        -i ${K8S_INVENTORY} \
                        playbooks/07.proxmox_k8s_docker_install.yml
                '''
            }
        }
        
        stage('☸️ Kubernetes componentas installation') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    # Install Kubernetes repository
                    ansible-playbook  \
                        -i ${K8S_INVENTORY} \
                        playbooks/08.proxmox_k8s_kube_repo.yml
                    # Install Kubernetes tools
                    ansible-playbook  \
                        -i ${K8S_INVENTORY} \
                        playbooks/09.proxmox_k8s_tools_setup.yml
                '''
            }
        }
        stage('🎮 Cluster Initialization') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook  \
                        -i ${K8S_INVENTORY} \
                        playbooks/10.proxmox_k8s_cluster_init.yml
                '''
            }
        }
        stage('🌐 Kubernetes network Setup') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook  \
                        -i ${K8S_INVENTORY} \
                        playbooks/11.proxmox_k8s_cni_install.yml
                '''
            }
        }
        stage('👥 Joining Worker Nodes') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook  \
                        -i ${K8S_INVENTORY} \
                        playbooks/12.proxmox_k8s_workers_join.yml
                '''
            }
        }
        stage('✅ Validating cluster setup') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    echo "Checking cluster status..."
                    # SSH to master and check cluster
                    ssh -o StrictHostKeyChecking=no lazarous@192.168.1.41 "kubectl get nodes -o wide" || echo "⚠️ Could not validate cluster"
                    ssh -o StrictHostKeyChecking=no lazarous@192.168.1.41 "kubectl get pods -A" || echo "⚠️ Could not get pod status"
                '''
            }
        }
        stage('🔄 VM Restart') {
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook  \
                        -i ${PROXMOX_INVENTORY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/13.proxmox_k8s_restart_vms.yml
                '''
            }
        }
    }
    
    post {
        always {
            script {
                def status = currentBuild.result ?: 'SUCCESS'
                echo "🏁 Deployment completed with status: ${status}"
                
                // Archive any generated files
                archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true
            }
        }
        success {
            echo "🎉 Kubernetes cluster deployment successful!"
            echo "📊 Check cluster status: kubectl get nodes"
        }
        failure {
            echo "❌ Deployment failed. Check the logs above for details."
            echo "🔍 Common issues: network connectivity, vault password, inventory configuration"
        }
        cleanup {
            echo "🧹 Cleaning up temporary files..."
            sh '''
                rm -f /tmp/kubeadm-join-command.sh || true
                rm -f ${VAULT_FILE} || true
            '''
        }
    }
}