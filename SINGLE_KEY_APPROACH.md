# Single SSH Key Approach - Simplified Solution

## Overview

The implementation now uses a **single SSH key pair** for both Proxmox host and VM access, simplifying key management and reducing complexity.

## Key Changes

### Previous Approach (Two Key Pairs)
```
~/.ssh/
├── jenkins_proxmox_key      # For Proxmox host
├── jenkins_proxmox_key.pub
├── jenkins_vm_key           # For VMs
└── jenkins_vm_key.pub
```

### New Approach (Single Key Pair)
```
~/.ssh/
├── jenkins_infra_key        # For BOTH Proxmox and VMs
└── jenkins_infra_key.pub
```

## Benefits

✅ **Simpler Management**
   - Only one key pair to generate
   - Only one key to distribute
   - Only one key to rotate

✅ **Reduced Complexity**
   - Fewer files to manage
   - Simpler vault configuration
   - Easier troubleshooting

✅ **Same Functionality**
   - Access to Proxmox host
   - Access to all VMs
   - Works in both Jenkins and local

✅ **Security Maintained**
   - Still uses ED25519 (modern, secure)
   - Proper file permissions (600)
   - Keys never committed to git

## How It Works

### 1. Key Generation

```bash
# Run playbook or script
./scripts/setup_ssh_keys.sh

# Generates single key pair
~/.ssh/jenkins_infra_key
~/.ssh/jenkins_infra_key.pub
```

### 2. Key Distribution

#### Proxmox Host
```bash
# Automatically or manually copy public key
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz
```

#### VMs
```yaml
# Cloud-init injects the same public key
users:
  - name: lazarous
    ssh_authorized_keys:
      - "{{ vault_ssh_public_key }}"
```

### 3. Usage

#### Connect to Proxmox
```bash
ssh -i ~/.ssh/jenkins_infra_key root@proxmox.laz
```

#### Connect to VMs
```bash
ssh -i ~/.ssh/jenkins_infra_key lazarous@192.168.1.41
ssh -i ~/.ssh/jenkins_infra_key lazarous@192.168.1.42
```

#### Ansible Usage
```yaml
# Inventory automatically uses the key
[proxmox]
proxmox.laz ansible_ssh_private_key_file={{ vault_ssh_private_key_path }}

[k8s_vms]
192.168.1.41 ansible_ssh_private_key_file={{ vault_ssh_private_key_path }}
```

## Vault Configuration

### Generated vault.yml

```yaml
# SSH Key Configuration (single key pair)
vault_ssh_private_key_path: "/path/to/.ssh/jenkins_infra_key"
vault_ssh_public_key_path: "/path/to/.ssh/jenkins_infra_key.pub"
vault_ssh_public_key: "ssh-ed25519 AAAAC3... jenkins-infra-12345"

# Proxmox Host Details
vault_proxmox_host: "proxmox.laz"
vault_proxmox_user: "root"

# VM Cloud-init Details
vault_ci_user: "lazarous"
vault_ci_password: "manun2001"
vault_ci_ssh_public_key_path: "/path/to/.ssh/jenkins_infra_key.pub"
vault_ci_ssh_private_key_path: "/path/to/.ssh/jenkins_infra_key"
```

**Note:** Both Proxmox and VM configurations reference the **same key**.

## Usage Examples

### Local Execution

```bash
# 1. Generate key
./scripts/setup_ssh_keys.sh

# 2. Copy to Proxmox (one-time setup)
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz

# 3. Run playbooks (VMs get key via cloud-init)
./scripts/run_playbooks.sh --yes

# 4. Access Proxmox
ssh -i ~/.ssh/jenkins_infra_key root@proxmox.laz

# 5. Access VMs
ssh -i ~/.ssh/jenkins_infra_key lazarous@192.168.1.41
```

### Jenkins Pipeline

```groovy
stage('🔑 Setup SSH Key') {
    steps {
        sh '''
            cd ${WORKSPACE}
            export SSH_KEY_DIR=${WORKSPACE}/.ssh
            ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
        '''
    }
}

stage('🏗️ Infrastructure Setup') {
    steps {
        sh '''
            # Single key used for all SSH connections
            ansible-playbook -i inventory/proxmox.ini \
                playbooks/02.proxmox_k8s_create_vm_template.yml
        '''
    }
}
```

## File Structure

### Local Environment
```
$HOME/.ssh/
├── jenkins_infra_key          # Private key (600)
└── jenkins_infra_key.pub      # Public key (644)

group_vars/vault.yml           # References jenkins_infra_key
```

### Jenkins Environment
```
$WORKSPACE/
├── .ssh/
│   ├── jenkins_infra_key      # Private key (600)
│   └── jenkins_infra_key.pub  # Public key (644)
└── group_vars/
    └── vault.yml              # References jenkins_infra_key
```

## Migration from Two Keys

If you have existing deployments with two separate keys, here's how to migrate:

### Option 1: Regenerate (Clean Slate)

```bash
# 1. Remove old keys
rm ~/.ssh/jenkins_proxmox_key*
rm ~/.ssh/jenkins_vm_key*

# 2. Generate new single key
./scripts/setup_ssh_keys.sh --regenerate

# 3. Copy to Proxmox
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz

# 4. Redeploy VMs (cloud-init will inject new key)
./scripts/run_playbooks.sh --yes
```

### Option 2: Manual Migration (Existing VMs)

```bash
# 1. Generate new single key
./scripts/setup_ssh_keys.sh

# 2. Copy to Proxmox
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz

# 3. Copy to existing VMs
for vm_ip in 192.168.1.41 192.168.1.42 192.168.1.43; do
    ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub lazarous@$vm_ip
done

# 4. Update vault.yml (automatic with playbook)
ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
```

## Security Considerations

### Why Single Key is Secure

1. **Separate Users:** Different users (root vs lazarous) provide separation
2. **Host-based Access:** Only authorized hosts can use the key
3. **Key Type:** ED25519 is modern and secure
4. **Permissions:** Proper file permissions prevent unauthorized access
5. **Rotation:** Easy to rotate single key when needed

### Key Rotation

```bash
# Generate new key
REGENERATE_KEYS=true ./scripts/setup_ssh_keys.sh

# Distribute to Proxmox
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz

# Redeploy VMs with new key
./scripts/run_playbooks.sh --yes
```

## Troubleshooting

### Key Not Working

```bash
# 1. Check key exists
ls -la ~/.ssh/jenkins_infra_key*

# 2. Check permissions
chmod 600 ~/.ssh/jenkins_infra_key
chmod 644 ~/.ssh/jenkins_infra_key.pub

# 3. Check vault.yml
cat group_vars/vault.yml | grep ssh

# 4. Test connection
ssh -i ~/.ssh/jenkins_infra_key -v root@proxmox.laz
```

### Multiple Keys Confusion

```bash
# List all jenkins keys
ls -la ~/.ssh/jenkins_*

# If old keys exist, remove them
rm ~/.ssh/jenkins_proxmox_key*
rm ~/.ssh/jenkins_vm_key*

# Keep only jenkins_infra_key
ls -la ~/.ssh/jenkins_infra_key*
```

### Jenkins Key Not Found

```bash
# In Jenkins workspace
echo $WORKSPACE
ls -la $WORKSPACE/.ssh/

# Verify key generation ran
cat $WORKSPACE/group_vars/vault.yml | grep ssh_private_key_path
```

## Comparison: Before vs After

### Before (Two Keys)

**Complexity:**
- Generate 2 key pairs
- Distribute 2 public keys
- Manage 2 sets of paths in vault
- Track which key for which purpose

**Files:**
```
~/.ssh/
├── jenkins_proxmox_key
├── jenkins_proxmox_key.pub
├── jenkins_vm_key
└── jenkins_vm_key.pub
```

### After (Single Key)

**Simplicity:**
- Generate 1 key pair
- Distribute 1 public key
- Manage 1 set of paths in vault
- One key for all access

**Files:**
```
~/.ssh/
├── jenkins_infra_key
└── jenkins_infra_key.pub
```

## Commands Reference

### Generate Key

```bash
# Standalone
./scripts/setup_ssh_keys.sh

# With playbooks
./scripts/run_playbooks.sh

# Force regenerate
./scripts/setup_ssh_keys.sh --regenerate

# Custom directory
./scripts/setup_ssh_keys.sh --ssh-dir /custom/path
```

### Distribute Key

```bash
# To Proxmox
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz

# To specific VM
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub lazarous@192.168.1.41

# To all VMs (loop)
for ip in 192.168.1.41 192.168.1.42 192.168.1.43; do
    ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub lazarous@$ip
done
```

### Use Key

```bash
# SSH to Proxmox
ssh -i ~/.ssh/jenkins_infra_key root@proxmox.laz

# SSH to VM
ssh -i ~/.ssh/jenkins_infra_key lazarous@192.168.1.41

# Run Ansible
ansible-playbook -i inventory/k8s_vms.ini playbook.yml
# (key path auto-loaded from vault.yml)
```

## Summary

The single key approach provides:

✅ **Simplicity:** One key for everything  
✅ **Ease of Use:** Fewer files to manage  
✅ **Same Security:** No security trade-offs  
✅ **Easy Rotation:** Rotate one key instead of two  
✅ **Clear Purpose:** `jenkins_infra_key` = infrastructure access  

**Result:** Simpler, cleaner, easier to maintain! 🎉

