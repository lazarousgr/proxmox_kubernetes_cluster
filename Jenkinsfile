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
        password(
            name: 'VAULT_PASSWORD',
            description: 'Ansible Vault password for decrypting vault.yml'
        )
    }
    
    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        ANSIBLE_STDOUT_CALLBACK = 'yaml'
        WORKSPACE_DIR = '/workspace'
        INVENTORY_FILE = '/workspace/inventory/k8s_vms.ini'
        HOSTS_INVENTORY = '/workspace/inventory/hosts.ini'
        VAULT_PASSWORD_FILE = '/tmp/vault_password'
    }
    
    stages {
        stage('🔍 Pre-flight Checks') {
            steps {
                script {
                    echo "🚀 Starting Kubernetes Cluster Deployment"
                    echo "Full Cluster Mode: ${params.FULL_CLUSTER}"
                    echo "Ansible Verbosity: ${params.ANSIBLE_VERBOSITY}"
                    
                    // Validate vault password parameter
                    if (!params.VAULT_PASSWORD) {
                        error "❌ VAULT_PASSWORD parameter is required"
                    }
                    
                    // Check workspace and inventory files
                    sh '''
                        echo "📂 Checking workspace structure..."
                        ls -la ${WORKSPACE_DIR}/
                        
                        if [ ! -d "${WORKSPACE_DIR}/playbooks" ]; then
                            echo "❌ Playbooks directory not found"
                            exit 1
                        fi
                        
                        if [ ! -f "${WORKSPACE_DIR}/group_vars/vault.yml" ]; then
                            echo "❌ Vault file not found"
                            exit 1
                        fi
                        
                        echo "✅ Workspace structure validated"
                    '''
                }
            }
        }
        
        stage('🔐 Vault Decryption') {
            steps {
                script {
                    echo "🔐 Handling Ansible Vault..."
                    sh '''
                        cd ${WORKSPACE_DIR}
                        
                        # Create temporary vault password file
                        echo "${VAULT_PASSWORD}" > ${VAULT_PASSWORD_FILE}
                        chmod 600 ${VAULT_PASSWORD_FILE}
                        
                        # Check if vault file is encrypted
                        if head -1 group_vars/vault.yml | grep -q "ANSIBLE_VAULT"; then
                            echo "🔒 Vault file is encrypted - decrypting..."
                            
                            # Decrypt vault file
                            ansible-vault decrypt group_vars/vault.yml --vault-password-file ${VAULT_PASSWORD_FILE}
                            
                            if [ $? -eq 0 ]; then
                                echo "✅ Vault file decrypted successfully"
                            else
                                echo "❌ Failed to decrypt vault file - check password"
                                exit 1
                            fi
                        else
                            echo "🔓 Vault file is already unencrypted"
                        fi
                    '''
                }
            }
        }
        
        stage('🏗️ Infrastructure Setup') {
            steps {
                echo "🏗️ Setting up infrastructure..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    
                    # Generate configurations
                    echo "📋 Generating inventory and configurations..."
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        playbooks/01.proxmox_k8s_generate_configs.yml
                    
                    # Create cloud template
                    echo "☁️ Creating VM template..."
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${HOSTS_INVENTORY} \
                        playbooks/02.proxmox_k8s_create_vm_template.yml
                    
                    # Clone VMs
                    echo "🖥️ Cloning VMs..."
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${HOSTS_INVENTORY} \
                        playbooks/03.proxmox_k8s_clone_vms.yml \
                        -e "deploy_workers=${params.FULL_CLUSTER}"
                    
                    # Start VMs
                    echo "🚀 Starting VMs..."
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${HOSTS_INVENTORY} \
                        playbooks/04.proxmox_k8s_start_vms.yml \
                        -e "deploy_workers=${params.FULL_CLUSTER}"
                '''
            }
        }
        
        stage('⚙️ System Preparation') {
            steps {
                echo "⚙️ Preparing systems for Kubernetes..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    
                    # System preparation (swap, sysctl, modules)
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${INVENTORY_FILE} \
                        playbooks/06.proxmox_k8s_os_prep.yml
                    
                    # Configure hostnames
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${INVENTORY_FILE} \
                        playbooks/05.proxmox_k8s_vms_hostname.yml
                    
                    # Install Docker/containerd
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${INVENTORY_FILE} \
                        playbooks/07.proxmox_k8s_docker_install.yml
                '''
            }
        }
        
        stage('☸️ Kubernetes Installation') {
            steps {
                echo "☸️ Installing Kubernetes components..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    
                    # Install Kubernetes repository
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${INVENTORY_FILE} \
                        playbooks/08.proxmox_k8s_kube_repo.yml
                    
                    # Install Kubernetes tools
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${INVENTORY_FILE} \
                        playbooks/09.proxmox_k8s_tools_setup.yml
                '''
            }
        }
        
        stage('🎮 Cluster Initialization') {
            steps {
                echo "🎮 Initializing Kubernetes cluster..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${INVENTORY_FILE} \
                        playbooks/10.proxmox_k8s_cluster_init.yml
                '''
            }
        }
        
        stage('�� Network Setup') {
            steps {
                echo "🌐 Installing CNI..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${INVENTORY_FILE} \
                        playbooks/11.proxmox_k8s_cni_install.yml
                '''
            }
        }
        
        stage('👥 Worker Nodes') {
            when { 
                params.FULL_CLUSTER == true 
            }
            steps {
                echo "👥 Joining worker nodes..."
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                        -i ${INVENTORY_FILE} \
                        playbooks/12.proxmox_k8s_workers_join.yml \
                        -e "deploy_workers={{ params.FULL_CLUSTER }}" || echo "⚠️ No worker nodes found or join failed"
                '''
            }
        }
        
        stage('✅ Cluster Validation') {
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
                rm -f ${VAULT_PASSWORD_FILE} || true
            '''
        }
    }
}