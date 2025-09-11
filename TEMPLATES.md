# 🎯 Template-Based Configuration System

This project uses a **template-based approach** to keep sensitive data secure while maintaining flexibility and reproducibility.

## 📁 **Directory Structure**

```
templates/                           # Template files (safe to commit)
├── group_vars/
│   └── proxmox.yml.j2              # Proxmox variables template
└── inventory/
    └── hosts.ini.j2                # Inventory template
scripts/                            # Generation scripts
├── generate_configs.py             # Python-based generator (gitignored)
└── generate_configs.sh             # Shell-based generator (gitignored)  
playbooks/
└── proxmox_k8s_generate_configs.yml # Ansible-based generator
group_vars/
└── vault.yml                       # Encrypted vault (gitignored)
inventory/                          # Generated files (gitignored)
├── hosts.ini                       # Generated inventory
└── group_vars/proxmox.yml          # Generated variables
```

## 🔐 **Vault Variables**

All sensitive data is stored in `group_vars/vault.yml`:

```yaml
# Proxmox connection details
vault_proxmox_host: "proxmox.example.com"
vault_proxmox_user: "root"
vault_ssh_private_key_path: "/path/to/private/key"

# VM user credentials  
vault_ci_user: "your_username"
vault_ci_password: "your_password"
vault_ci_ssh_public_key_path: "/path/to/public/key"

# Template VM configuration
vault_template_vm_id: 9000

# K8s VM specifications
vault_k8s_hosts:
  - id: 8001
    name: "kmaster"
    ip: "192.168.1.41"
    mac: "08:00:27:D5:26:51"
    memory: 4096
    cores: 2
  - id: 8002
    name: "kworker1"
    ip: "192.168.1.42"
    mac: "08:00:27:D5:26:52"
    memory: 3072
    cores: 1
```

## 🛠️ **Generation Methods**

### **Method 1: Ansible (Recommended)**
```bash
# Generate all config files from templates
ansible-playbook playbooks/proxmox_k8s_generate_configs.yml
```

### **Method 2: Python Script** (if available)
```bash
# Requires: pip install jinja2 pyyaml
./scripts/generate_configs.py
```

### **Method 3: Shell Script** (if available)
```bash
./scripts/generate_configs.sh
```

### **Method 4: Setup Script (Automated)**
```bash
# Automatically chooses best available method
./setup.sh
```

## 🎨 **Template Format**

### **Jinja2 Templates (.j2 files)**

**Variables:**
```yaml
# Use double braces for variables
storage: "{{ vault_storage_name }}"
ci_user: "{{ vault_ci_user }}"
```

**Conditionals:**
```yaml
# Optional disk resizing
{% if disk_resize_enable %}
disk_size: "{{ vault_disk_size }}"
{% endif %}
```

**Loops:**
```yaml
# Generate multiple entries
{% for host in vault_k8s_hosts %}
{{ host.ip }} ansible_user={{ vault_ci_user }}
{% endfor %}
```

### **Example Template Usage**

**templates/inventory/hosts.ini.j2:**
```ini
[proxmox]
{{ vault_proxmox_host }} ansible_user={{ vault_proxmox_user }} ansible_ssh_private_key_file={{ vault_ssh_private_key_path }}

[k8s_nodes]
{% for host in vault_k8s_hosts %}
{{ host.ip }} ansible_user={{ vault_ci_user }} ansible_ssh_private_key_file={{ vault_ssh_private_key_path }}
{% endfor %}
```

**templates/group_vars/proxmox.yml.j2:**
```yaml
---
template_vm_id: "{{ vault_template_vm_id }}"
storage: "{{ vault_storage }}"
disk_size: "{{ vault_disk_size }}"
disk_resize_enable: {{ disk_resize_enable | default(True) }}
ci_user: "{{ vault_ci_user }}"
ci_password: "{{ vault_ci_password }}"
```

## 🔄 **Workflow**

1. **Edit templates** in `templates/` directory
2. **Update vault** with new variables: `nano group_vars/vault.yml`
3. **Generate configs**: `ansible-playbook playbooks/proxmox_k8s_generate_configs.yml`
4. **Use generated files**: Run your playbooks with generated inventory
5. **Commit only templates**: Generated files are automatically gitignored

## 🚫 **What Gets Ignored by Git**

```gitignore
# Generated configuration files (contain real values)
inventory/hosts.ini
inventory/group_vars/proxmox.yml
group_vars/vault.yml

# Generation scripts (contain sensitive logic)
scripts/*

# Templates are safe to commit (contain only placeholders)
templates/
```

## ✅ **Security Benefits**

- ✨ **No sensitive data in repo history** - All credentials in gitignored vault
- 🔐 **Encrypted credentials** - Can optionally encrypt vault with ansible-vault
- 📋 **Templates provide clear structure** - Easy to see what variables are needed
- 🔄 **Easy to regenerate configs** - One command updates all files
- 👥 **Team collaboration** - Everyone can use their own values
- 🎯 **Environment-specific** - Different values for dev/staging/production

## 🔧 **Advanced Usage**

### **Conditional Generation**
```yaml
# In template
{% if environment == "production" %}
memory: 8192
{% else %}
memory: 4096
{% endif %}
```

### **Variable Defaults**
```yaml
# In template  
disk_size: "{{ vault_disk_size | default('20G') }}"
cores: {{ vault_cores | default(2) }}
```

### **Multiple Environments**
```bash
# Different vault files per environment
ansible-playbook playbooks/proxmox_k8s_generate_configs.yml -e vault_file=group_vars/vault_production.yml
```

### **Override Variables**
```bash
# Override template variables at runtime
ansible-playbook playbooks/proxmox_k8s_generate_configs.yml -e disk_resize_enable=False
```

## 📋 **Template Best Practices**

1. **Use descriptive variable names** with `vault_` prefix for sensitive data
2. **Include comments** explaining what each section does
3. **Provide defaults** for optional variables
4. **Group related variables** logically in the vault
5. **Test templates** with example data before using real credentials
6. **Document required variables** in template comments
7. **Use consistent formatting** across all templates

This template system ensures your sensitive data stays secure while making it easy to maintain and deploy your infrastructure! 🎉
