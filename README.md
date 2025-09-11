# 🚀 Proxmox Kubernetes Cluster Automation

## Overview

This repository contains Ansible playbooks to **automate the creation and configuration of a Kubernetes cluster** within a Proxmox environment.  
It leverages cloud-init for VM provisioning, template-based configuration generation, and supports full lifecycle management from VM creation to Kubernetes installation.

---

## 🗂️ Project Structure

```
proxmox_kubernetes_cluster/
├── playbooks/
│   ├── proxmox_k8s_create_vm_template.yml    # Create Ubuntu cloud template VM in Proxmox
│   ├── proxmox_k8s_clone_vms.yml             # Clone multiple VMs from template
│   ├── proxmox_k8s_generate_configs.yml      # Generate configs from templates
│   └── k8s_vm_config.yml                     # Configure VMs for Kubernetes
├── templates/
│   ├── group_vars/proxmox.yml.j2             # Proxmox variables template
│   └── inventory/hosts.ini.j2                # Inventory template
├── group_vars/
│   └── vault.yml                             # Encrypted sensitive variables (gitignored)
├── inventory/                                # Generated inventory files (gitignored)
│   ├── hosts.ini                             # Generated from template
│   └── group_vars/proxmox.yml                # Generated from template
├── scripts/
│   ├── generate_configs.py                   # Python config generator
│   └── generate_configs.sh                   # Shell config generator
└── setup.sh                                  # Complete setup script
```

---

## 🛠️ Features

- 🖥️ **Automated VM Template Creation**  
  Build a cloud-init enabled Ubuntu template for rapid VM provisioning with automatic cleanup.

- 🧬 **VM Cloning with Custom Hardware**  
  Clone multiple VMs with unique MAC addresses, disk sizes, memory, and CPU configurations.

- � **Template-Based Security**  
  All sensitive data (IPs, credentials) stored in encrypted vault, with templates generating actual config files.

- 🎯 **Conditional Operations**  
  Disk resizing and other operations can be enabled/disabled via configuration flags.

- 🏗️ **Kubernetes-Ready Configuration**  
  Installs Docker, kubelet, kubeadm, kubectl, and disables swap on target VMs.

---

## � Quick Start

### 1. **Initial Setup**
```bash
# Clone the repository
git clone <your-repo-url>
cd proxmox_kubernetes_cluster

# Run setup script (creates example vault if needed)
./setup.sh
```

### 2. **Configure Your Environment**
```bash
# Edit vault with your actual values
nano group_vars/vault.yml

# Optionally encrypt the vault (recommended for production)
ansible-vault encrypt group_vars/vault.yml
```

### 3. **Generate Configuration Files**
```bash
# Generate inventory and config files from templates
ansible-playbook playbooks/proxmox_k8s_generate_configs.yml
```

### 4. **Create VM Template**
```bash
# Create Ubuntu cloud-init template on Proxmox
ansible-playbook playbooks/proxmox_k8s_create_vm_template.yml
```

### 5. **Clone VMs for Kubernetes**
```bash
# Clone VMs from template with custom specifications
ansible-playbook playbooks/proxmox_k8s_clone_vms.yml
```

### 6. **Configure VMs for Kubernetes**
```bash
# Install and configure Kubernetes components
ansible-playbook -i inventory/hosts.ini playbooks/k8s_vm_config.yml
```

---

## 📋 Configuration

### **Vault Variables (group_vars/vault.yml)**
```yaml
# Proxmox connection
vault_proxmox_host: "proxmox.example.com"
vault_proxmox_user: "root"
vault_ssh_private_key_path: "/path/to/private/key"

# VM user credentials
vault_ci_user: "your_username"
vault_ci_password: "your_password"
vault_ci_ssh_public_key_path: "/path/to/public/key"

# Template configuration
vault_template_vm_id: 9000

# VM specifications
vault_k8s_hosts:
  - id: 8001
    name: "kmaster"
    ip: "192.168.1.41"
    mac: "08:00:27:D5:26:51"
    memory: 4096
    cores: 2
```

### **Infrastructure Settings (inventory/group_vars/proxmox.yml)**
```yaml
# Generated from template - do not edit directly
storage: "ThinVault"
disk_size: "30G"
disk_resize_enable: True
image_url: "https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"
```

---

## 🔐 Security

- **Sensitive data encrypted** in Ansible Vault
- **Generated files gitignored** (contain real credentials/IPs)
- **Template-based approach** keeps repository clean
- **Conditional operations** prevent accidental changes

See [SECURITY.md](SECURITY.md) for detailed security practices.

---

## 📚 Documentation

- [TEMPLATES.md](TEMPLATES.md) - Template system and config generation
- [SECURITY.md](SECURITY.md) - Security best practices and vault management
- [USAGE_EXAMPLES.md](USAGE_EXAMPLES.md) - Detailed usage examples and workflows

---

## 🔧 Advanced Usage

### **Custom Kubernetes Version**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/k8s_vm_config.yml -e k8s_version=v1.29
```

### **Disable Disk Resizing**
```yaml
# In templates/group_vars/proxmox.yml.j2
disk_resize_enable: False
```

### **Override VM Specifications**
```bash
ansible-playbook playbooks/proxmox_k8s_clone_vms.yml -e '{"vault_k8s_hosts":[{"id":8001,"name":"test","ip":"192.168.1.100","mac":"08:00:27:D5:26:99","memory":2048,"cores":1}]}'
```

---

## 🧑‍💻 Requirements

- **Proxmox VE 7.0+** with SSH access
- **Ansible 2.9+** 
- **Python 3.6+** (for script-based config generation)
- **wget** (for cloud image download)
- **Cloud-init enabled template support**

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes (update templates, not generated files)
4. Test with your environment
5. Submit a pull request

---

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.