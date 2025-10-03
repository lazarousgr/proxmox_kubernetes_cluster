# 🔐 SSH Key Authentication Workflow

This document explains the two-phase SSH authentication workflow used in this Proxmox Kubernetes cluster deployment.

## 🎯 **Overview**

The deployment uses a **two-phase authentication approach**:
1. **Phase 1**: Generate configurations with password authentication
2. **Phase 2**: Install SSH keys, then regenerate configurations with key authentication

This approach ensures secure, automated deployment while maintaining the ability to bootstrap SSH key authentication.

## 📁 **Template Files**

### **Password Authentication Template**
```jinja2
# templates/inventory/proxmox.ini_with_ssh_passwd.j2
[proxmox]
{{ vault_proxmox_host }} ansible_user={{ vault_proxmox_user }} ansible_password={{ lookup('env','PROXMOX_PASS') }}
```

### **SSH Key Authentication Template**
```jinja2
# templates/inventory/proxmox.ini_with_ssh_key.j2
[proxmox]
{{ vault_proxmox_host }} ansible_user={{ vault_proxmox_user }} ansible_ssh_private_key_file={{ vault_ssh_private_key_path }}
```

### **Dynamic Template (Alternative)**
```jinja2
# templates/inventory/proxmox.ini.j2
[proxmox]
{% if use_ssh_key | default(false) %}
{{ vault_proxmox_host }} ansible_user={{ vault_proxmox_user }} ansible_ssh_private_key_file={{ vault_ssh_private_key_path }}
{% else %}
{{ vault_proxmox_host }} ansible_user={{ vault_proxmox_user }} ansible_password={{ lookup('env','PROXMOX_PASS') }}
{% endif %}
```

## 🚀 **Jenkins Pipeline Workflow**

### **Stage 1: SSH Key Generation**
```groovy
stage('🔐 Generate SSH Keys') {
    steps {
        echo "🔐 Generating SSH keys..."
        sh """
            cd ${WORKSPACE_DIR}
            export SSH_KEY_DIR=${WORKSPACE_DIR}/.ssh
            ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                playbooks/00.proxmox_k8s_generate_ssh_keys_merge.yml
        """
    }
}
```

### **Stage 2: SSH Key Installation**
```groovy
stage('🔑 Install SSH Keys on Proxmox') {
    steps {
        // 1. Generate configs with password authentication
        ansible-playbook playbooks/01a.proxmox_k8s_generate_configs_with_password.yml
        
        // 2. Install SSH key using ssh-copy-id
        sshpass -p '${params.PROXMOX_PASSWORD}' ssh-copy-id \
            -o StrictHostKeyChecking=no \
            -i ${WORKSPACE_DIR}/.ssh/jenkins_infra_key.pub \
            root@proxmox.laz
        
        // 3. Regenerate configs with SSH key authentication
        ansible-playbook playbooks/01b.proxmox_k8s_generate_configs_with_key.yml
    }
}
```

## 📋 **Playbook Structure**

### **Phase 1: Password Authentication**
- **File**: `playbooks/01a.proxmox_k8s_generate_configs_with_password.yml`
- **Template**: `proxmox.ini_with_ssh_passwd.j2`
- **Purpose**: Generate initial configurations for SSH key installation

### **Phase 2: SSH Key Authentication**
- **File**: `playbooks/01b.proxmox_k8s_generate_configs_with_key.yml`
- **Template**: `proxmox.ini_with_ssh_key.j2`
- **Purpose**: Generate production configurations with SSH key authentication

## 🔧 **Configuration Variables**

### **Vault Variables** (`group_vars/vault.yml`)
```yaml
# Proxmox connection details
vault_proxmox_host: "proxmox.laz"
vault_proxmox_user: "root"
vault_ssh_private_key_path: "/home/lazarous/.ssh/id_ed25519"

# VM user credentials  
vault_ci_user: "lazarous"
vault_ci_password: "manun2001"
vault_ci_ssh_public_key_path: "/home/lazarous/.ssh/id_ed25519.pub"
```

### **Jenkins Parameters**
```groovy
parameters {
    password(
        name: 'PROXMOX_PASSWORD',
        description: 'Proxmox root password for initial SSH key installation',
        defaultValue: ''
    )
}
```

## 🔄 **Workflow Steps**

1. **🔐 Generate SSH Keys**
   - Create SSH key pair in Jenkins workspace
   - Store keys in `${WORKSPACE_DIR}/.ssh/`

2. **📋 Generate Password Configs**
   - Use `01a.proxmox_k8s_generate_configs_with_password.yml`
   - Generate `proxmox.ini` with password authentication
   - Use Jenkins `PROXMOX_PASSWORD` parameter

3. **🔑 Install SSH Keys**
   - Use `sshpass` and `ssh-copy-id` to install public key
   - Target: `root@proxmox.laz`
   - Key: `${WORKSPACE_DIR}/.ssh/jenkins_infra_key.pub`

4. **🔄 Regenerate Key Configs**
   - Use `01b.proxmox_k8s_generate_configs_with_key.yml`
   - Generate `proxmox.ini` with SSH key authentication
   - Ready for production deployment

5. **🏗️ Infrastructure Deployment**
   - Use SSH key-authenticated configurations
   - Deploy VMs, Kubernetes, etc.

## 🛡️ **Security Benefits**

- **No Hardcoded Passwords**: Password comes from Jenkins parameter
- **Temporary Password Use**: Password only used for initial key installation
- **Key-Based Production**: All subsequent operations use SSH keys
- **Secure Key Storage**: Keys stored in Jenkins workspace with proper permissions

## 🚨 **Troubleshooting**

### **SSH Key Installation Fails**
```bash
# Check if sshpass is installed
which sshpass

# Test password authentication manually
sshpass -p 'PASSWORD' ssh -o StrictHostKeyChecking=no root@proxmox.laz

# Check SSH key format
ssh-keygen -l -f ${WORKSPACE_DIR}/.ssh/jenkins_infra_key.pub
```

### **Template Generation Issues**
```bash
# Test template generation manually
ansible-playbook playbooks/01a.proxmox_k8s_generate_configs_with_password.yml -v

# Check generated inventory
cat inventory/proxmox.ini
```

### **Permission Issues**
```bash
# Fix SSH key permissions
chmod 600 ${WORKSPACE_DIR}/.ssh/jenkins_infra_key
chmod 644 ${WORKSPACE_DIR}/.ssh/jenkins_infra_key.pub
```

## 📝 **Usage Examples**

### **Manual Deployment**
```bash
# Phase 1: Generate with password
export PROXMOX_PASS="your_password"
ansible-playbook playbooks/01a.proxmox_k8s_generate_configs_with_password.yml

# Install SSH key
sshpass -p 'your_password' ssh-copy-id -i .ssh/jenkins_infra_key.pub root@proxmox.laz

# Phase 2: Generate with key
ansible-playbook playbooks/01b.proxmox_k8s_generate_configs_with_key.yml

# Deploy infrastructure
ansible-playbook -i inventory/proxmox.ini playbooks/02.proxmox_k8s_create_vm_template.yml
```

### **Jenkins Pipeline**
The Jenkins pipeline automatically handles both phases:
1. User provides `PROXMOX_PASSWORD` parameter
2. Pipeline generates password configs
3. Pipeline installs SSH keys
4. Pipeline regenerates key configs
5. Pipeline continues with infrastructure deployment

## ✅ **Validation**

After successful deployment, verify:
- SSH key authentication works: `ssh -i .ssh/jenkins_infra_key root@proxmox.laz`
- Ansible can connect: `ansible proxmox -i inventory/proxmox.ini -m ping`
- No password prompts during deployment
- All subsequent operations use SSH keys
