# SSH Key Setup Guide

This guide explains how SSH keys are managed for both **local execution** and **Jenkins (containerized)** environments.

## Overview

The project uses **automated SSH key generation** that works seamlessly in both environments:
- **Local Execution:** Keys stored in `$HOME/.ssh`
- **Jenkins (Containerized):** Keys stored in `$WORKSPACE/.ssh`

## Quick Start

### For Local Execution

```bash
# Generate SSH keys automatically
./scripts/setup_ssh_keys.sh

# Or run playbooks directly (keys auto-generated)
./scripts/run_playbooks.sh --yes

# Force regenerate keys
./scripts/setup_ssh_keys.sh --regenerate
```

### For Jenkins

```groovy
// In your Jenkinsfile
stage('🔑 Generate SSH Keys') {
    steps {
        sh '''
            cd ${WORKSPACE}
            export SSH_KEY_DIR=${WORKSPACE}/.ssh
            export REGENERATE_KEYS=false
            ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
        '''
    }
}
```

## How It Works

### 1. Automatic Environment Detection

The playbook `00.proxmox_k8s_generate_ssh_keys.yml` automatically detects the execution environment:

```yaml
# Detects if running in Jenkins
is_jenkins: "{{ lookup('env', 'JENKINS_HOME') | default('', true) != '' }}"

# Sets SSH key directory accordingly
ssh_key_dir: 
  - Jenkins: "$WORKSPACE/.ssh"
  - Local: "$HOME/.ssh"
```

### 2. SSH Key Generation

Two key pairs are generated:
- **Proxmox Key:** For Jenkins/Local → Proxmox host SSH access
- **VM Key:** For Jenkins/Local → VM SSH access

Keys are named:
- `jenkins_proxmox_key` / `jenkins_proxmox_key.pub`
- `jenkins_vm_key` / `jenkins_vm_key.pub`

### 3. Vault Configuration Update

The `group_vars/vault.yml` file is automatically updated with SSH key paths:

```yaml
# Proxmox Host Details
vault_ssh_private_key_path: "/path/to/.ssh/jenkins_proxmox_key"
vault_ssh_public_key_path: "/path/to/.ssh/jenkins_proxmox_key.pub"

# VM Cloud-init Details
vault_ci_ssh_private_key_path: "/path/to/.ssh/jenkins_vm_key"
vault_ci_ssh_public_key_path: "/path/to/.ssh/jenkins_vm_key.pub"
```

## Usage Patterns

### Pattern 1: Local Development

```bash
# 1. Generate SSH keys
./scripts/setup_ssh_keys.sh

# 2. Copy Proxmox public key to Proxmox host
ssh-copy-id -i ~/.ssh/jenkins_proxmox_key.pub root@proxmox.laz

# 3. Run playbooks
./scripts/run_playbooks.sh --yes
```

### Pattern 2: Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    environment {
        SSH_KEY_DIR = "${WORKSPACE}/.ssh"
        REGENERATE_KEYS = "false"
    }
    
    stages {
        stage('🔑 Setup SSH Keys') {
            steps {
                sh '''
                    ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
                '''
            }
        }
        
        stage('🏗️ Run Infrastructure Setup') {
            steps {
                sh '''
                    # Keys are now available in $WORKSPACE/.ssh
                    ansible-playbook -i inventory/proxmox.ini \
                        playbooks/02.proxmox_k8s_create_vm_template.yml
                '''
            }
        }
    }
}
```

### Pattern 3: Custom SSH Directory

```bash
# Local with custom directory
./scripts/setup_ssh_keys.sh --ssh-dir /custom/path/.ssh

# Or via environment variable
SSH_KEY_DIR=/custom/path/.ssh ./scripts/run_playbooks.sh --yes
```

## Configuration Options

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `SSH_KEY_DIR` | `$HOME/.ssh` or `$WORKSPACE/.ssh` | SSH key directory |
| `REGENERATE_KEYS` | `false` | Force regenerate existing keys |
| `JENKINS_HOME` | Auto-detected | Jenkins environment detection |
| `WORKSPACE` | Auto-detected | Jenkins workspace directory |

### Script Options

#### setup_ssh_keys.sh

```bash
./scripts/setup_ssh_keys.sh [OPTIONS]

Options:
  --regenerate            Force regeneration of existing keys
  --ssh-dir DIR           Override SSH key directory
  --help                  Show help message
```

#### run_playbooks.sh

```bash
./scripts/run_playbooks.sh [OPTIONS]

Options:
  --ssh-dir DIR           Override SSH key directory
  --regenerate-keys       Regenerate SSH keys before running
  --skip-key-gen          Skip SSH key generation
  --yes                   Skip all prompts
  --help                  Show help message
```

## Ansible Playbook Integration

### Direct Playbook Execution

```bash
# Generate keys with custom directory
SSH_KEY_DIR=/custom/.ssh \
REGENERATE_KEYS=true \
ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml

# Check generated keys
ls -la /custom/.ssh/jenkins_*
```

### In Other Playbooks

Keys are automatically available via vault variables:

```yaml
- name: Connect to Proxmox
  hosts: proxmox
  vars:
    ansible_ssh_private_key_file: "{{ vault_ssh_private_key_path }}"
  tasks:
    - name: Run command on Proxmox
      command: pvesm status
```

## File Locations

### Local Execution

```
$HOME/.ssh/
├── jenkins_proxmox_key        # Proxmox private key
├── jenkins_proxmox_key.pub    # Proxmox public key
├── jenkins_vm_key             # VM private key
└── jenkins_vm_key.pub         # VM public key

group_vars/vault.yml           # Updated with key paths
```

### Jenkins Execution

```
$WORKSPACE/
├── .ssh/
│   ├── jenkins_proxmox_key
│   ├── jenkins_proxmox_key.pub
│   ├── jenkins_vm_key
│   └── jenkins_vm_key.pub
├── group_vars/
│   └── vault.yml              # Updated with key paths
└── playbooks/
    └── ...
```

## Troubleshooting

### Keys Not Found

```bash
# Check environment detection
ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml -v

# Verify SSH_KEY_DIR
echo $SSH_KEY_DIR

# List generated keys
ls -la ${SSH_KEY_DIR:-$HOME/.ssh}/jenkins_*
```

### Permission Denied

```bash
# Fix SSH directory permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/jenkins_*
```

### Keys Not Working in Jenkins

```bash
# Verify keys exist in workspace
ls -la $WORKSPACE/.ssh/

# Check vault.yml paths
cat $WORKSPACE/group_vars/vault.yml | grep ssh

# Test SSH connection
ssh -i $WORKSPACE/.ssh/jenkins_proxmox_key root@proxmox.laz "hostname"
```

### Regenerate Keys

```bash
# Local
./scripts/setup_ssh_keys.sh --regenerate

# Jenkins
REGENERATE_KEYS=true ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
```

## Security Best Practices

1. **Key Rotation:** Regenerate keys periodically
   ```bash
   ./scripts/setup_ssh_keys.sh --regenerate
   ```

2. **Key Permissions:** Always ensure proper permissions
   ```bash
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/jenkins_*
   ```

3. **Jenkins Secrets:** Store keys in Jenkins credentials for production
4. **Gitignore:** SSH keys are git-ignored by default
5. **Cleanup:** Keys are automatically cleaned up in Jenkins post-build

## Advanced Usage

### Multiple Environments

```bash
# Development
SSH_KEY_DIR=~/.ssh/dev ./scripts/run_playbooks.sh --yes

# Production
SSH_KEY_DIR=~/.ssh/prod ./scripts/run_playbooks.sh --yes
```

### Key Fingerprint Verification

```bash
# View key fingerprints
ssh-keygen -lf ~/.ssh/jenkins_proxmox_key.pub
ssh-keygen -lf ~/.ssh/jenkins_vm_key.pub
```

### Manual Key Distribution

```bash
# Copy Proxmox key to Proxmox host
ssh-copy-id -i ~/.ssh/jenkins_proxmox_key.pub root@proxmox.laz

# Copy VM key to specific VM
ssh-copy-id -i ~/.ssh/jenkins_vm_key.pub lazarous@192.168.1.41
```

## Summary

This unified approach provides:

✅ **Automatic environment detection** (Jenkins vs Local)  
✅ **Flexible key storage** (workspace vs home directory)  
✅ **Automated key generation** (no manual intervention)  
✅ **Seamless integration** (with both scripts and playbooks)  
✅ **Security best practices** (proper permissions, key rotation)  

The system "just works" in both environments without code changes!

