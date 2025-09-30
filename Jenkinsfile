pipeline {
    agent any
    
    parameters {
        booleanParam(
            name: 'FULL_CLUSTER',
            defaultValue: true,
            description: 'Deploy full cluster (master + workers) or single node only'
        )
        choice(
            name: 'ANSIBLE_VERBOSITY',
            choices: ['-v', '-vv', '-vvv', ''],
            description: 'Ansible verbosity level'
        )
        credentials(
            name: 'VAULT_FILE_CREDENTIAL',
            credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.FileCredentialsImpl',
            description: 'Vault configuration file',
            required: true
        )
    }
    
    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        ANSIBLE_STDOUT_CALLBACK = 'yaml'
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
                    echo "Full Cluster Mode: ${params.FULL_CLUSTER}"
                    echo "Ansible Verbosity: ${params.ANSIBLE_VERBOSITY}"
                    
                    // Validate vault file credential
                    if (!params.VAULT_FILE_CREDENTIAL) {
                        error "❌ VAULT_FILE_CREDENTIAL parameter is required"
                    }
                    
                    // Check workspace and inventory files
                    sh '''
                        echo "📂 Checking workspace structure..."
                        ls -la ${WORKSPACE_DIR}/
                        
                        if [ ! -d "${WORKSPACE_DIR}/playbooks" ]; then
                            echo "❌ Playbooks directory not found"
                            exit 1
                        fi
                        
                        # if [ ! -f "${WORKSPACE_DIR}/group_vars/vault.yml" ]; then
                        #     echo "❌ Vault file not found"
                        #     exit 1
                        # fi
                        
                        echo "✅ Workspace structure validated"
                    '''
                }
            }
        }
        
        stage('🔐 Extract Vault Configuration') {
            steps {
                script {
                    echo "🔐 Extracting vault configuration from Jenkins..."
                    withCredentials([file(credentialsId: params.VAULT_FILE_CREDENTIAL, variable: 'VAULT_FILE_PATH')]) {
                        sh '''
                            cd ${WORKSPACE_DIR}
                            
                            # Create group_vars directory if it doesn't exist
                            mkdir -p group_vars
                            
                            # Copy vault file from Jenkins credential
                            cp ${VAULT_FILE_PATH} ${VAULT_FILE}
                            
                            # Set secure permissions
                            chmod 600 ${VAULT_FILE}
                            
                            echo "✅ Vault configuration extracted successfully"
                        '''
                    }
                }
            }
        }
        
        stage('🏗️ Infrastructure Setup') {
            steps {
                echo "🏗️ Setting up infrastructure..."
                sh """
                    cd ${WORKSPACE_DIR}
                    
                    # Generate configurations
                    echo "📋 Generating inventory and configurations..."
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        playbooks/01.proxmox_k8s_generate_configs.yml \
                        -e "include_workers=${params.FULL_CLUSTER}"
                    
                    # Create cloud template
                    echo "☁️ Creating VM template..."
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${PROXMOX_INVENTORY} \
                        playbooks/02.proxmox_k8s_create_vm_template.yml
                    
                    # Clone VMs
                    echo "🖥️ Cloning VMs..."
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${PROXMOX_INVENTORY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/03.proxmox_k8s_clone_vms.yml
                    
                    # Start VMs
                    echo "🚀 Starting VMs..."
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${PROXMOX_INVENTORY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/04.proxmox_k8s_start_vms.yml
                """
            }
        }
        
        stage('⚙️ System Preparation') {
            steps {
                echo "⚙️ Preparing systems for Kubernetes..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    
                    # System preparation (swap, sysctl, modules)
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/06.proxmox_k8s_os_prep.yml
                    
                    # Configure hostnames
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/05.proxmox_k8s_vms_hostname.yml
                    
                    # Install Docker/containerd
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/07.proxmox_k8s_docker_install.yml
                '''
            }
        }
        
        stage('☸️ Kubernetes Installation') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                echo "☸️ Installing Kubernetes components..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    # Install Kubernetes repository
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/08.proxmox_k8s_kube_repo.yml
                    # Install Kubernetes tools
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
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
                echo "🎮 Initializing Kubernetes cluster..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/10.proxmox_k8s_cluster_init.yml
                '''
            }
        }
        stage('🌐 Network Setup') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                echo "🌐 Installing CNI..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/11.proxmox_k8s_cni_install.yml
                '''
            }
        }
        stage('👥 Worker Nodes') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                echo "👥 Joining worker nodes..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${K8S_INVENTORY} \
                        playbooks/12.proxmox_k8s_workers_join.yml
                '''
            }
        }
        stage('✅ Cluster Validation') {
            when {
                expression { params.FULL_CLUSTER == true }
            }
            steps {
                echo "✅ Validating cluster setup..."
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
                echo "🔄 Restarting all VMs..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
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