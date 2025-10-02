# SSH Key Workflow - Visual Guide

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                   EXECUTION ENVIRONMENT                          │
├─────────────────────────────┬───────────────────────────────────┤
│      LOCAL WORKSTATION      │      JENKINS CONTAINER            │
│                             │                                   │
│  User runs:                 │  Pipeline stage runs:             │
│  ./scripts/run_playbooks.sh │  ansible-playbook ...             │
│                             │                                   │
│  SSH Keys Location:         │  SSH Keys Location:               │
│  $HOME/.ssh/                │  $WORKSPACE/.ssh/                 │
│  ├── jenkins_proxmox_key    │  ├── jenkins_proxmox_key          │
│  ├── jenkins_proxmox_key.pub│  ├── jenkins_proxmox_key.pub      │
│  ├── jenkins_vm_key         │  ├── jenkins_vm_key               │
│  └── jenkins_vm_key.pub     │  └── jenkins_vm_key.pub           │
│                             │                                   │
│  Vault Location:            │  Vault Location:                  │
│  ./group_vars/vault.yml     │  $WORKSPACE/group_vars/vault.yml  │
└─────────────────────────────┴───────────────────────────────────┘
```

## Execution Flow

### Option 1: Using run_playbooks.sh

```
┌─────────────────────────────────────────────────────────────────┐
│                 ./scripts/run_playbooks.sh --yes                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Environment Detection                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  if JENKINS_HOME exists:                                  │  │
│  │    → Jenkins mode: SSH_KEY_DIR=$WORKSPACE/.ssh           │  │
│  │  else:                                                    │  │
│  │    → Local mode: SSH_KEY_DIR=$HOME/.ssh                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: SSH Key Generation                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Run: playbooks/00.proxmox_k8s_generate_ssh_keys.yml     │  │
│  │                                                           │  │
│  │  Creates:                                                 │  │
│  │  - $SSH_KEY_DIR/jenkins_proxmox_key                      │  │
│  │  - $SSH_KEY_DIR/jenkins_proxmox_key.pub                  │  │
│  │  - $SSH_KEY_DIR/jenkins_vm_key                           │  │
│  │  - $SSH_KEY_DIR/jenkins_vm_key.pub                       │  │
│  │                                                           │  │
│  │  Updates:                                                 │  │
│  │  - group_vars/vault.yml (with key paths)                 │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Run Infrastructure Playbooks                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  01. Generate configs                                     │  │
│  │  02. Create VM template (uses Proxmox key)              │  │
│  │  03. Clone VMs                                           │  │
│  │  04. Start VMs                                           │  │
│  │  05. Setup SSH on VMs (distributes VM key)              │  │
│  │  06-13. Additional setup (uses VM key)                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Option 2: Using setup_ssh_keys.sh

```
┌─────────────────────────────────────────────────────────────────┐
│              ./scripts/setup_ssh_keys.sh [OPTIONS]               │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Only generates SSH keys and updates vault.yml                   │
│  Does NOT run infrastructure playbooks                           │
│                                                                  │
│  Use this when:                                                  │
│  - You only need to generate/regenerate keys                     │
│  - You want to test key generation                              │
│  - You want to inspect generated keys before using              │
└─────────────────────────────────────────────────────────────────┘
```

## SSH Key Distribution

```
┌─────────────────────────────────────────────────────────────────┐
│                        LOCAL WORKSTATION                         │
│                                                                  │
│  ~/.ssh/jenkins_proxmox_key.pub                                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               │ ssh-copy-id (manual or automated)
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        PROXMOX HOST                              │
│                                                                  │
│  /root/.ssh/authorized_keys                                     │
│  └── Contains: jenkins_proxmox_key.pub content                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   VM TEMPLATE CLOUD-INIT                         │
│                                                                  │
│  Cloud-init config includes:                                     │
│  - vault_ci_ssh_public_key (jenkins_vm_key.pub)                │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               │ VM cloning propagates key
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     CLONED VMs (K8s Nodes)                       │
│                                                                  │
│  /home/lazarous/.ssh/authorized_keys                            │
│  └── Contains: jenkins_vm_key.pub content                       │
└─────────────────────────────────────────────────────────────────┘
```

## Authentication Flow

### Proxmox Access

```
┌──────────────┐                               ┌──────────────┐
│   Jenkins    │  ssh -i jenkins_proxmox_key   │   Proxmox    │
│   or Local   │ ──────────────────────────────▶│     Host     │
│              │  root@proxmox.laz             │              │
└──────────────┘                               └──────────────┘
      │                                               │
      │ Uses private key:                             │ Verifies with:
      │ $SSH_KEY_DIR/jenkins_proxmox_key             │ /root/.ssh/authorized_keys
      │                                               │ (contains public key)
      └───────────────────────────────────────────────┘
```

### VM Access

```
┌──────────────┐                               ┌──────────────┐
│   Jenkins    │  ssh -i jenkins_vm_key        │  K8s Master  │
│   or Local   │ ──────────────────────────────▶│  or Worker   │
│              │  lazarous@192.168.1.41        │     VM       │
└──────────────┘                               └──────────────┘
      │                                               │
      │ Uses private key:                             │ Verifies with:
      │ $SSH_KEY_DIR/jenkins_vm_key                  │ /home/lazarous/.ssh/authorized_keys
      │                                               │ (contains public key)
      └───────────────────────────────────────────────┘
```

## Ansible Integration

### Inventory Configuration (Auto-generated)

```ini
# inventory/proxmox.ini
[proxmox]
proxmox.laz ansible_ssh_private_key_file={{ vault_ssh_private_key_path }} ansible_ssh_user=root

# inventory/k8s_vms.ini
[k8s_masters]
192.168.1.41 ansible_ssh_private_key_file={{ vault_ci_ssh_private_key_path }} ansible_ssh_user=lazarous

[k8s_workers]
192.168.1.42 ansible_ssh_private_key_file={{ vault_ci_ssh_private_key_path }} ansible_ssh_user=lazarous
192.168.1.43 ansible_ssh_private_key_file={{ vault_ci_ssh_private_key_path }} ansible_ssh_user=lazarous
```

### Playbook Execution

```
┌─────────────────────────────────────────────────────────────────┐
│  ansible-playbook -i inventory/proxmox.ini playbook.yml          │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Ansible reads inventory                                         │
│  └── Gets ansible_ssh_private_key_file from vault               │
│      (vault_ssh_private_key_path)                               │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Ansible loads vault.yml                                         │
│  └── vault_ssh_private_key_path = $SSH_KEY_DIR/jenkins_key     │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Ansible connects using the key                                  │
│  └── ssh -i $SSH_KEY_DIR/jenkins_key user@host                  │
└─────────────────────────────────────────────────────────────────┘
```

## Jenkins Pipeline Example

```groovy
pipeline {
    agent any
    
    environment {
        WORKSPACE_DIR = "${WORKSPACE}"
        SSH_KEY_DIR = "${WORKSPACE}/.ssh"
        REGENERATE_KEYS = "false"
    }
    
    stages {
        stage('🔑 Setup SSH Keys') {
            steps {
                sh '''
                    # Generate SSH keys
                    ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
                    
                    # Verify keys
                    ls -la ${SSH_KEY_DIR}/
                '''
            }
        }
        
        stage('🏗️ Infrastructure Setup') {
            steps {
                sh '''
                    # Keys are now available for use
                    ansible-playbook -i inventory/proxmox.ini \
                        playbooks/02.proxmox_k8s_create_vm_template.yml
                '''
            }
        }
        
        stage('☸️ Kubernetes Setup') {
            steps {
                sh '''
                    # Use VM keys for K8s nodes
                    ansible-playbook -i inventory/k8s_vms.ini \
                        playbooks/10.proxmox_k8s_cluster_init.yml
                '''
            }
        }
    }
    
    post {
        cleanup {
            sh '''
                # Cleanup ephemeral keys
                rm -rf ${SSH_KEY_DIR}
            '''
        }
    }
}
```

## Decision Tree

```
Start
  │
  ├─ Running in Jenkins?
  │  ├─ YES → Use $WORKSPACE/.ssh
  │  └─ NO  → Use $HOME/.ssh
  │
  ├─ Keys exist?
  │  ├─ YES → Use existing (unless --regenerate)
  │  └─ NO  → Generate new keys
  │
  ├─ Need to distribute keys?
  │  ├─ Proxmox → ssh-copy-id (manual or automated)
  │  └─ VMs     → Cloud-init (automated)
  │
  └─ Ready to run playbooks!
```

## Comparison: Before vs After

### Before (Manual)

```
❌ Manual key generation
❌ Hard-coded paths in vault.yml
❌ Different paths for Jenkins vs Local
❌ Manual key distribution
❌ Path mismatches cause failures
```

### After (Automated)

```
✅ Automatic key generation
✅ Dynamic paths in vault.yml
✅ Works in both Jenkins and Local
✅ Automated key distribution
✅ Environment-aware configuration
```

## Summary

This implementation provides:

1. **Environment Detection:** Automatically detects Jenkins vs Local
2. **Key Generation:** Generates keys in the correct location
3. **Configuration Update:** Updates vault.yml with correct paths
4. **Key Distribution:** Distributes keys to Proxmox and VMs
5. **Seamless Integration:** Works with all existing playbooks

**Result:** A single codebase that works everywhere! 🎉

