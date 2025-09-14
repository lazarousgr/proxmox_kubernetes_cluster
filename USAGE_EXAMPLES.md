# 📝 Usage Examples and Workflows

This document provides comprehensive examples of how to use the Proxmox Kubernetes Cluster automation system.

## 🚀 **Complete Workflow Examples**

### **Example 1: Full New Environment Setup**

```bash
# 1. Initial setup
git clone <your-repository>
cd proxmox_kubernetes_cluster
./setup.sh

# 2. Configure your environment
nano group_vars/vault.yml
# Add your Proxmox details, credentials, VM specifications

# 3. Generate configuration files from templates
ansible-playbook playbooks/proxmox_k8s_generate_configs.yml

# 4. Create VM template in Proxmox
ansible-playbook playbooks/proxmox_k8s_create_vm_template.yml

# 5. Clone VMs for Kubernetes cluster
ansible-playbook playbooks/proxmox_k8s_clone_vms.yml

# 6. Configure VMs for Kubernetes
ansible-playbook -i inventory/hosts.ini playbooks/k8s_vm_config.yml
```

### **Example 2: Development Environment**

```bash
# Quick setup for testing
./setup.sh
nano group_vars/vault.yml  # Set smaller VM specs for development

# Generate configs
ansible-playbook playbooks/proxmox_k8s_generate_configs.yml

# Create template with different settings
ansible-playbook playbooks/proxmox_k8s_create_vm_template.yml -e template_vm_id=9001

# Clone smaller VMs
ansible-playbook playbooks/proxmox_k8s_clone_vms.yml -e disk_resize_enable=False
```

## 🔧 **Configuration Examples**

### **Vault Configuration Examples**

**Basic Single-Node Setup:**
```yaml
# group_vars/vault.yml
# Proxmox connection
vault_proxmox_host: "proxmox.example.com"
vault_proxmox_user: "admin_user"
vault_ssh_private_key_path: "/path/to/private/key"

# VM user credentials
vault_ci_user: "your_username"
vault_ci_password: "your_password"
vault_ci_ssh_public_key_path: "/path/to/public/key"

# Template configuration
vault_template_vm_id: proxmox_vm_id

# Kubernetes cluster configuration
vault_k8s_masters:
  - ip: "x.x.x.x"
    name: "k8s-master-01"
    id: proxmox_vm_id_for_master_node
    mac: "xx:xx:xx:xx:xx:xx"
    memory: 4096
    cores: 2
    disk_size: "10G"

# Kubernetes configuration
kubernetes_version: "1.30"
pod_network_cidr: "10.244.0.0/16"
service_subnet: "10.96.0.0/12" # Not used for dev env
```

**Production 3-Node Cluster:**
```yaml
# group_vars/vault.yml
# Proxmox connection
vault_proxmox_host: "proxmox.example.com"
vault_proxmox_user: "admin_user"
vault_ssh_private_key_path: "/path/to/private/key"

# VM user credentials
vault_ci_user: "your_username"
vault_ci_password: "your_password"
vault_ci_ssh_public_key_path: "/path/to/public/key"

# Template configuration
vault_template_vm_id: proxmox_vm_id

# Kubernetes cluster configuration
vault_k8s_masters:
  - ip: "x.x.x.x"
    name: "k8s-master-01"
    id: proxmox_vm_id_for_master_node
    mac: "xx:xx:xx:xx:xx:xx"
    memory: 4096
    cores: 2
    disk_size: "10G"

vault_k8s_workers:
  - ip: "x.x.x.x"
    name: "k8s-worker-01"
    id: proxmox_vm_id_for_worker_node1
    mac: "xx:xx:xx:xx:xx:xx"
    memory: 3072
    cores: 1
    disk_size: "10G"
  - ip: "x.x.x.x"
    name: "k8s-worker-02"
    id: proxmox_vm_id_for_worker_node2
    mac: "xx:xx:xx:xx:xx:xx"
    memory: 3072
    cores: 1
    disk_size: "10G"

# Kubernetes configuration
kubernetes_version: "1.30"
pod_network_cidr: "10.244.0.0/16"
service_subnet: "10.96.0.0/12" # Not used for dev env
```

## 📋 **Playbook Usage Examples**

### **Template Creation with Custom Settings**

```bash
# Create template with specific Ubuntu version
ansible-playbook playbooks/proxmox_k8s_create_vm_template.yml \
  -e image_url="https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img"

# Create template with different storage
ansible-playbook playbooks/proxmox_k8s_create_vm_template.yml \
  -e storage="fast-ssd"
```

### **VM Cloning with Variations**

```bash
# Clone VMs without disk resizing
ansible-playbook playbooks/proxmox_k8s_clone_vms.yml \
  -e disk_resize_enable=False

# Clone VMs with different disk size
ansible-playbook playbooks/proxmox_k8s_clone_vms.yml \
  -e disk_size="50G"
```

### **Kubernetes Configuration Examples**

```bash
# Install specific Kubernetes version
ansible-playbook -i inventory/hosts.ini playbooks/k8s_vm_config.yml \
  -e k8s_version="v1.28"

# Configure only master nodes
ansible-playbook -i inventory/hosts.ini playbooks/k8s_vm_config.yml \
  --limit "192.168.1.101"
```

## 🛠️ **Troubleshooting Examples**

### **Template Creation Issues**

```bash
# Check if template already exists
ansible -i inventory/hosts.ini proxmox -m shell -a "qm list | grep vm_id"

# Remove existing template if needed
ansible -i inventory/hosts.ini proxmox -m shell -a "qm destroy vm_id"
```

### **VM Creation Issues**

```bash
# Check if VM IDs are available
ansible -i inventory/hosts.ini proxmox -m shell -a "qm list"

# Verify cloud-init configuration
ansible -i inventory/hosts.ini proxmox -m shell -a "qm config vm_id"
```

### **Network Connectivity Issues**

```bash
# Test SSH connectivity to VMs
ansible -i inventory/hosts.ini k8s_nodes -m ping

# Check if cloud-init completed
ansible -i inventory/hosts.ini k8s_nodes -m shell -a "cloud-init status"
```

## 📊 **Monitoring and Validation**

### **Verify Deployment Success**

```bash
# Check all VMs are running
ansible proxmox -m shell -a "qm list | grep running"

# Verify Kubernetes installation
ansible k8s_cluster -m shell -a "kubectl version --client"

# Check Docker status
ansible k8s_cluster -m shell -a "systemctl status containerd" --become
```

This comprehensive examples guide should help you use the automation system effectively in various scenarios! 🎉