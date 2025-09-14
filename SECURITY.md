# 🔐 Security and Credentials Management

This project implements a comprehensive security strategy using Ansible Vault, template-based configuration, and Git exclusion patterns to protect sensitive information.

## 🛡️ **Security Architecture**

### **Three-Layer Security Approach:**

1. **Git Exclusion** - Sensitive files never committed to repository
2. **Template System** - Only placeholder templates in version control  
3. **Vault Encryption** - Optional additional encryption for sensitive data

## 📁 **File Security Classification**

| **Security Level** | **Files** | **Git Status** | **Contains** |
|-------------------|-----------|----------------|--------------|
| 🟢 **Safe** | `templates/`, `playbooks/` | ✅ Committed | Placeholder variables only |
| 🟡 **Sensitive** | `group_vars/vault.yml` | ❌ Gitignored | Real credentials, IPs |
| 🟡 **Generated** | `inventory/`, `scripts/` | ❌ Gitignored | Real configuration data |
| 🔴 **Secrets** | `.vault_pass`, SSH keys | ❌ Gitignored | Authentication credentials |

## 🔐 **Vault Configuration**

### **Required Vault Variables**
```yaml
# group_vars/vault.yml
# Proxmox connection
vault_proxmox_host: "proxmox"
vault_proxmox_user: "root"
vault_ssh_private_key_path: "/path/to/private/key"

# VM credentials
vault_ci_user: "your_username"
vault_ci_password: "your_secure_password"
vault_ci_ssh_public_key_path: "/path/to/public/key"

# Infrastructure
vault_template_vm_id: "your_template_id"
vault_storage: "your_storage_name"
vault_disk_size: "your_preferred_disk_size"

# VM specifications
vault_k8s_hosts:
  - id: disk
    name: "kmaster"
    ip: "192.168.1.41"
    mac: "08:00:27:D5:26:51"
    memory: 4096
    cores: 2
```

## 🔒 **Ansible Vault Operations**

### **Basic Vault Management**

```bash
# Create new encrypted vault
ansible-vault create group_vars/vault.yml

# Edit encrypted vault
ansible-vault edit group_vars/vault.yml

# View encrypted vault without editing
ansible-vault view group_vars/vault.yml

# Encrypt existing file
ansible-vault encrypt group_vars/vault.yml

# Decrypt file (temporary)
ansible-vault decrypt group_vars/vault.yml

# Change vault password
ansible-vault rekey group_vars/vault.yml
```

## 🎯 **Deployment Security Strategies**

### **Development Environment**
```bash
# Keep vault unencrypted for easy editing
# Use .gitignore to prevent commits
echo "group_vars/vault.yml" >> .gitignore
```

### **Production Environment**
```bash
# Always encrypt vault
ansible-vault encrypt group_vars/vault.yml

# Use secure password storage
# Store vault password in secure password manager
```

### **CI/CD Environment**
```bash
# Use environment variables for vault password
export ANSIBLE_VAULT_PASSWORD_FILE=/path/to/secure/password/file

# Mount secrets as files or environment variables
# Never hardcode credentials in CI configuration
```

## 🛠️ **Alternative Security Methods**

### **Environment Variables Only**
```yaml
# In playbooks, reference environment variables directly
vault_ci_user: "{{ ansible_env.CI_USER | default('user') }}"
vault_ci_password: "{{ ansible_env.CI_PASSWORD | default('') }}"
```

```bash
# Set environment variables
export CI_USER="your_username"
export CI_PASSWORD="your_password"
ansible-playbook playbooks/proxmox_k8s_create_vm_template.yml
```

### **External Secret Management**
```yaml
# Integration with HashiCorp Vault, AWS Secrets Manager, etc.
vault_ci_password: "{{ lookup('community.hashi_vault.hashi_vault', 'secret=secret/data/ci_password') }}"
```

## 🔍 **Security Validation**

### **Pre-commit Checks**
```bash
# Check for accidentally committed secrets
git log --oneline | head -10
git show --name-only

# Verify .gitignore is working
git status --ignored
```

### **Repository Scanning**
```bash
# Scan for potential secrets in history
git log --all --full-history -- group_vars/vault.yml

# Check current repository state
git ls-files | grep -E "(vault|password|key|secret)"
```

## 🔧 **Development Workflow Security**

```bash
# 1. Initial setup
git clone <repository>
cd proxmox_kubernetes_cluster
cp group_vars/vault.yml.example group_vars/vault.yml
nano group_vars/vault.yml  # Add your values

# 2. Generate configurations
ansible-playbook playbooks/proxmox_k8s_generate_configs.yml

# 3. Verify no sensitive data will be committed
git status
git diff --cached

# 4. Safe commit (only templates and playbooks)
git add templates/ playbooks/ *.md
git commit -m "Update templates and documentation"
```

This comprehensive security approach ensures your infrastructure automation remains secure while maintaining usability and collaboration capabilities! 🛡️
