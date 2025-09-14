# 🚀 Proxmox Kubernetes Cluster Automation

## Overview

This repository contains Ansible playbooks to **automate the complete creation and configuration of a dev-ready Kubernetes cluster** within a Proxmox environment. It provides end-to-end automation from VM template creation to a fully configured Kubernetes control plane with worker nodes ready to join.

The project uses a **sequential numbered playbook approach** for clear workflow management, template-based configuration generation, and comprehensive logging.

This project was triggered when I had to set up a local kubernetes env on m proxmox for the 10th time and I realized that this is a pure waste of time and I should automate it end to end.

---

## 🗂️ Project Structure

```
proxmox_kubernetes_cluster/
├── playbooks/                               # Sequential numbered playbooks
│   ├── 01.proxmox_k8s_generate_configs.yml  # Generate inventory and configs
│   ├── 02.proxmox_k8s_cloud_template.yml    # Create Ubuntu cloud template
│   ├── 03.proxmox_k8s_clone_vms.yml         # Clone VMs from template
│   ├── 04.proxmox_k8s_dock_kube_prep.yml    # System preparation (swap, sysctl, modules)
│   ├── 05.proxmox_k8s_vms_hostname.yml      # Configure hostnames and /etc/hosts
│   ├── 06.proxmox_k8s_docker_install.yml    # Install and configure containerd
│   ├── 07.proxmox_k8s_kube_install.yml      # Install Kubernetes components
│   └── 08.proxmox_k8s_controlplane_setup.yml # Initialize Kubernetes cluster
├── templates/
│   ├── group_vars/proxmox.yml.j2            # Proxmox variables template
│   ├── inventory/hosts.ini.j2               # Proxmox hosts inventory template
│   └── inventory/k8s_vms.ini.j2             # Kubernetes cluster inventory template
├── group_vars/
│   └── vault.yml                            # Encrypted sensitive variables
├── inventory/                              # Generated inventory files
│   ├── hosts.ini                           # Generated Proxmox hosts inventory
│   ├── k8s_vms.ini                         # Generated Kubernetes cluster inventory
│   └── group_vars/proxmox.yml              # Generated Proxmox variables
├── scripts/
│   ─── run_playbooks.sh                    # Execute all playbooks 
└── ansible.cfg                             # Ansible configuration
```

---

## 🛠️ Features

- 🖥️ **Automated VM Template Creation**  
  Build a cloud-init enabled Ubuntu template for rapid VM provisioning with automatic cleanup.

- 🧬 **VM Cloning with Custom Hardware**  
  Clone multiple VMs with unique MAC addresses, disk sizes, memory, and CPU configurations as per Kubernetes requirements.

- 🔒 **Template-Based Security**  
  All sensitive data (IPs, credentials) stored in encrypted vault, with templates generating actual config files.

- 🎯 **System Preparation**  
  Comprehensive VM preparation including swap disable, kernel modules, sysctl settings for Kubernetes.

- 🏗️ **Complete Kubernetes Installation**  
  Installs containerd, kubelet, kubeadm, kubectl with proper CRI configuration and version holding.

- 🌐 **Hostname & Network Configuration**  
  Sets proper hostnames and configures /etc/hosts for all cluster nodes for better name resolution.

- � **Control Plane Initialization**  
  Automated Kubernetes cluster initialization with join command generation for worker nodes.

- 📊 **Comprehensive Logging**  
  Detailed progress tracking and status reporting throughout the entire deployment process.

- 🔄 **Sequential Workflow**  
  Numbered playbooks ensure proper execution order and clear dependency management.

---

## ⚡ Quick Start

### 1. **Complete Automated Setup**
```bash
# Clone the repository
git clone <your-repo-url>
cd proxmox_kubernetes_cluster

# Edit vault.yml to match your k8s setup. Sample file provided
./group_vars/vault.yml

# Run all playbooks in sequence (recommended)
./scripts/run_playbooks.sh
```

### 2. **Step-by-Step Setup**

#### **Initial Configuration**
```bash
# Edit vault with your actual values
ansible-vault create group_vars/vault.yml
# or
ansible-vault edit group_vars/vault.yml
```

#### **Generate Configuration Files**
```bash
# Generate inventory and config files from templates
ansible-playbook playbooks/01.proxmox_k8s_generate_configs.yml --ask-vault-pass
```

#### **Create VM Infrastructure**
```bash
# Create Ubuntu vm template on Proxmox
ansible-playbook playbooks/02.proxmox_k8s_create_vm_template.yml --ask-vault-pass

# Clone VMs from template with custom specifications
ansible-playbook playbooks/03.proxmox_k8s_clone_vms.yml --ask-vault-pass

# Start VMs from template with custom specifications
ansible-playbook playbooks/04.proxmox_k8s_start_vms.yml --ask-vault-pass

# Configure hostnames and /etc/hosts
ansible-playbook playbooks/05.proxmox_k8s_vms_hostname.yml --ask-vault-pass
```

#### **Prepare VMs for Kubernetes**
```bash
# System preparation (swap, sysctl, modules)
ansible-playbook playbooks/06.proxmox_k8s_os_prep.yml --ask-vault-pass
```

#### **Install Container Runtime and Kubernetes Tools**
```bash
# Install and configure containerd
ansible-playbook playbooks/07.proxmox_k8s_docker_install.yml --ask-vault-pass

# Install Kubernetes repository
ansible-playbook playbooks/08.proxmox_k8s_kube_repo.yml --ask-vault-pass

# Initialize Kubernetes cluster
ansible-playbook playbooks/09.proxmox_k8s_tools_setup.yml --ask-vault-pass
```
#### **Initialize Cluster**
```bash
# Initialize cluster on master node
ansible-playbook playbooks/10.proxmox_k8s_cluster_init.yml --ask-vault-pass

# Install CNI
ansible-playbook playbooks/11.proxmox_k8s_cni_install.yml --ask-vault-pass

# Joine worker nodes to the cluster
ansible-playbook playbooks/12.proxmox_k8s_workers_join.yml --ask-vault-pass
```

## 📋 Configuration

### **Vault Variables (group_vars/vault.yml)**
```yaml
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

### **Infrastructure Settings (inventory/group_vars/proxmox.yml)**
```yaml
# Generated from template - do not edit directly
storage: "local-lvm"
disk_size: "30G"
disk_resize_enable: true # If true it will resize the vm disks as per disk_size spec
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

### **Individual Playbook Execution**
```bash
# Run specific playbooks individually
ansible-playbook playbooks/04.proxmox_k8s_dock_kube_prep.yml --ask-vault-pass
ansible-playbook playbooks/05.proxmox_k8s_vms_hostname.yml --ask-vault-pass
```

### **Custom Kubernetes Version**
```bash
# Edit vault.yml to change kubernetes_version
ansible-vault edit group_vars/vault.yml
# Change: kubernetes_version: "1.29"
```

### **Disable Serial Execution**
```yaml
# In any playbook, change:
serial: 1  # to process VMs one at a time
# to:
# serial: "50%"  # or remove for parallel execution
```

### **Environment-Specific Inventories**
```bash
# Use different inventory files for different environments
ansible-playbook -i inventory/production/k8s_cluster.ini playbooks/04.proxmox_k8s_dock_kube_prep.yml
ansible-playbook -i inventory/staging/k8s_cluster.ini playbooks/04.proxmox_k8s_dock_kube_prep.yml
```

### **Override VM Specifications**
```bash
# Override vault variables at runtime
ansible-playbook playbooks/03.proxmox_k8s_clone_vms.yml -e '{"vault_k8s_masters":[{"id":8001,"name":"test-master","ip":"192.168.1.100","mac":"08:00:27:D5:26:99","memory":2048,"cores":1}]}'
```

### **Skip Certain Tasks**
```bash
# Skip specific tags
ansible-playbook playbooks/04.proxmox_k8s_dock_kube_prep.yml --skip-tags "swap,kernel_modules"
```

---

## 🧑‍💻 Requirements

- **Proxmox VE 7.0+** with SSH access
- **Ansible 2.9+** 
- **Python 3.6+** (for script-based config generation)
- **wget** (for cloud image download)
- **Cloud-init enabled template support**

---

## 🔄 Workflow

The complete workflow follows this sequence:

1. **Configuration Generation** - Templates generate inventory files with real IPs/hostnames
2. **Template Creation** - VM template with cloud-init support (Ubuntu)
3. **VM Cloning** - Multiple VMs cloned with unique specifications
4. **Boot VMs** - Initial boot and cloud init activities
5. **Hostname Configuration** - Proper hostnames and DNS resolution setup
6. **System Preparation** - Kernel modules, sysctl settings, swap configuration
7. **Container Runtime** - containerd installation with Kubernetes-compatible configuration
8. **Kubernetes Repository** - Add Kubernetes repository
9. **Kubernetes Installation** - kubelet, kubeadm, kubectl with version holding
10. **Cluster Initialization** - Cluster init on master node
11. **CNI Installation** - Flannel network plugin for pod-to-pod communication
12. **Worker Join** - Worker nodes join the cluster and become ready

