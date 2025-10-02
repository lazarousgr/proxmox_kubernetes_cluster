# Quick Reference - SSH Key Management

## TL;DR

**Single command for both local and Jenkins:**
```bash
./scripts/setup_ssh_keys.sh
```

**It just works!** The script auto-detects the environment and does the right thing.

---

## Environment Detection

### Automatic Detection

```yaml
# In playbooks/00.proxmox_k8s_generate_ssh_keys.yml
is_jenkins: "{{ lookup('env', 'JENKINS_HOME') | default('', true) != '' }}"

# If JENKINS_HOME exists → Jenkins mode → Keys in $WORKSPACE/.ssh
# If JENKINS_HOME empty   → Local mode   → Keys in $HOME/.ssh
```

### Jenkins Variables (Auto-Provided)

| Variable | Example | Always Available? |
|----------|---------|------------------|
| `JENKINS_HOME` | `/var/jenkins_home` | ✅ Yes |
| `WORKSPACE` | `/var/jenkins_home/workspace/job` | ✅ Yes |
| `BUILD_NUMBER` | `42` | ✅ Yes |

---

## Key Locations

### Local Execution
```
$HOME/.ssh/
├── jenkins_infra_key         # Private key
└── jenkins_infra_key.pub     # Public key
```

### Jenkins Execution
```
$WORKSPACE/.ssh/
├── jenkins_infra_key         # Private key
└── jenkins_infra_key.pub     # Public key
```

---

## Quick Commands

### Generate Keys

```bash
# Local
./scripts/setup_ssh_keys.sh

# Jenkins (in pipeline)
export SSH_KEY_DIR=${WORKSPACE}/.ssh
ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
```

### Distribute Keys

```bash
# To Proxmox
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz

# VMs get key automatically via cloud-init
```

### Use Keys

```bash
# Connect to Proxmox
ssh -i ~/.ssh/jenkins_infra_key root@proxmox.laz

# Connect to VMs
ssh -i ~/.ssh/jenkins_infra_key lazarous@192.168.1.41
```

### Regenerate Keys

```bash
# Local
./scripts/setup_ssh_keys.sh --regenerate

# Jenkins
export REGENERATE_KEYS=true
ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
```

---

## Run Playbooks

### Local
```bash
# With automatic key generation
./scripts/run_playbooks.sh --yes

# Skip key generation
./scripts/run_playbooks.sh --skip-key-gen --yes

# Regenerate keys
./scripts/run_playbooks.sh --regenerate-keys --yes
```

### Jenkins
```groovy
stage('🔑 SSH Keys') {
    steps {
        sh 'ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml'
    }
}

stage('🏗️ Infrastructure') {
    steps {
        sh 'ansible-playbook -i inventory/proxmox.ini playbooks/02.proxmox_k8s_create_vm_template.yml'
    }
}
```

---

## Vault Configuration

### Auto-Generated vault.yml

```yaml
# Single SSH key for both Proxmox and VMs
vault_ssh_private_key_path: "/path/.ssh/jenkins_infra_key"
vault_ssh_public_key_path: "/path/.ssh/jenkins_infra_key.pub"
vault_ssh_public_key: "ssh-ed25519 AAAAC3... jenkins-infra-12345"

# Proxmox uses this key
vault_proxmox_user: "root"

# VMs also use this key
vault_ci_user: "lazarous"
vault_ci_ssh_private_key_path: "{{ vault_ssh_private_key_path }}"
```

---

## Environment Variables

### Set These for Custom Behavior

| Variable | Default | Purpose |
|----------|---------|---------|
| `SSH_KEY_DIR` | `$HOME/.ssh` or `$WORKSPACE/.ssh` | Override key location |
| `REGENERATE_KEYS` | `false` | Force regenerate keys |
| `JENKINS_HOME` | Auto-detected | Jenkins environment flag |
| `WORKSPACE` | Auto-detected | Jenkins workspace path |

### Examples

```bash
# Custom key directory
SSH_KEY_DIR=/custom/path ./scripts/setup_ssh_keys.sh

# Force regenerate
REGENERATE_KEYS=true ./scripts/setup_ssh_keys.sh

# Simulate Jenkins environment
JENKINS_HOME=/var/jenkins_home WORKSPACE=/tmp/workspace ./scripts/setup_ssh_keys.sh
```

---

## Troubleshooting

### Keys Not Found

```bash
# Check key location
ls -la ~/.ssh/jenkins_infra_key*
# or
ls -la $WORKSPACE/.ssh/jenkins_infra_key*

# Check vault configuration
cat group_vars/vault.yml | grep ssh
```

### Permission Denied

```bash
# Fix permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/jenkins_infra_key
chmod 644 ~/.ssh/jenkins_infra_key.pub
```

### Wrong Environment Detected

```bash
# Check environment variables
echo "JENKINS_HOME: ${JENKINS_HOME}"
echo "WORKSPACE: ${WORKSPACE}"

# Force local mode
unset JENKINS_HOME
./scripts/setup_ssh_keys.sh

# Force Jenkins mode
export JENKINS_HOME=/var/jenkins_home
export WORKSPACE=/tmp/test
./scripts/setup_ssh_keys.sh
```

### Keys Not Working

```bash
# Test Proxmox connection
ssh -i ~/.ssh/jenkins_infra_key -v root@proxmox.laz

# Test VM connection
ssh -i ~/.ssh/jenkins_infra_key -v lazarous@192.168.1.41

# Check authorized_keys on remote
ssh root@proxmox.laz "cat ~/.ssh/authorized_keys"
ssh lazarous@192.168.1.41 "cat ~/.ssh/authorized_keys"
```

---

## File Structure

```
project/
├── playbooks/
│   └── 00.proxmox_k8s_generate_ssh_keys.yml  # Key generation
├── scripts/
│   ├── setup_ssh_keys.sh                     # Standalone script
│   └── run_playbooks.sh                      # Full deployment
├── templates/
│   └── vault.yml.j2                          # Vault template
├── group_vars/
│   └── vault.yml                             # Generated config
├── .ssh/                                      # Generated keys (local)
│   ├── jenkins_infra_key
│   └── jenkins_infra_key.pub
└── Documentation/
    ├── SSH_KEY_SETUP.md                      # Full guide
    ├── JENKINS_ENV_VARS.md                   # Jenkins env guide
    ├── SINGLE_KEY_APPROACH.md                # Single key explanation
    ├── KEY_COMPARISON.md                     # Before/after comparison
    └── QUICK_REFERENCE.md                    # This file
```

---

## Decision Tree

```
Need SSH keys?
├─ Yes
│  ├─ Running locally?
│  │  └─ ./scripts/setup_ssh_keys.sh
│  │     → Keys in ~/.ssh/jenkins_infra_key
│  │
│  └─ Running in Jenkins?
│     └─ ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
│        → Keys in $WORKSPACE/.ssh/jenkins_infra_key
│
└─ No
   └─ Keys already exist, continue with playbooks
```

---

## One-Liners

```bash
# Setup everything (local)
./scripts/setup_ssh_keys.sh && ./scripts/run_playbooks.sh --yes

# Check key fingerprint
ssh-keygen -lf ~/.ssh/jenkins_infra_key.pub

# Copy key to Proxmox
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz

# Test all connections
for host in root@proxmox.laz lazarous@192.168.1.41 lazarous@192.168.1.42; do
    ssh -i ~/.ssh/jenkins_infra_key -o ConnectTimeout=2 $host hostname
done

# Clean up all jenkins keys
rm -f ~/.ssh/jenkins_*
```

---

## Complete Jenkins Pipeline Example

```groovy
pipeline {
    agent any
    environment {
        WORKSPACE_DIR = "${WORKSPACE}"
        SSH_KEY_DIR = "${WORKSPACE}/.ssh"
    }
    stages {
        stage('🔑 SSH') {
            steps {
                sh 'ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml'
            }
        }
        stage('🏗️ Infra') {
            steps {
                sh 'ansible-playbook -i inventory/proxmox.ini playbooks/02.proxmox_k8s_create_vm_template.yml'
            }
        }
    }
    post {
        cleanup {
            sh 'rm -rf ${SSH_KEY_DIR}'
        }
    }
}
```

---

## Help Commands

```bash
# Setup script help
./scripts/setup_ssh_keys.sh --help

# Run playbooks help
./scripts/run_playbooks.sh --help

# Ansible playbook help
ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml --help
```

---

## Documentation Links

| Document | Purpose |
|----------|---------|
| `SSH_KEY_SETUP.md` | Complete setup guide |
| `JENKINS_ENV_VARS.md` | Jenkins environment variables |
| `SINGLE_KEY_APPROACH.md` | Why single key is better |
| `KEY_COMPARISON.md` | Before/after comparison |
| `QUICK_REFERENCE.md` | This file (quick lookup) |

---

**Remember:** The system auto-detects everything. Just run the scripts! 🚀

