# 🚀 Proxmox Kubernetes Cluster Automation

## Overview

This repository contains Ansible playbooks to **automate the creation and configuration of a Kubernetes cluster** within my Proxmox environment.  
It leverages cloud-init for VM provisioning, native Ansible modules for guest configuration, and supports full lifecycle management from VM creation to Kubernetes installation.

---

## 🗂️ Project Structure

```
kubernetes_proxmox/
├── playbooks/
│   ├── proxmox_ubuntu_template.yml      # Create Ubuntu cloud template VM in Proxmox
│   ├── proxmox_clone_vms.yml            # Clone multiple VMs from template
│   ├── k8s_vm_config.yml                # Configure VMs for Kubernetes (Docker, kubeadm, etc.)
├── group_vars/
│   └── proxmox.yml                      # Variables for Proxmox hosts
├── inventory/
│   ├── proxmox_hosts.ini                # Proxmox inventory file
│   └── k8s_vms.ini                      # Inventory for Kubernetes VM nodes
├── cloudinit/
│   └── user-data.yaml                   # Cloud-init config for passwordless sudo
└── README.md
```

---

## 🛠️ Features

- 🖥️ **Automated VM Template Creation**  
  Build a cloud-init enabled Ubuntu template for rapid VM provisioning.

- 🧬 **VM Cloning with Custom Hardware**  
  Clone multiple VMs with unique MAC addresses, disk sizes, and static IPs.

- 🔑 **Cloud-Init Integration**  
  Set SSH keys, users, and passwordless sudo for instant Ansible access.

- 🏗️ **Kubernetes-Ready Configuration**  
  Installs Docker, kubelet, kubeadm, kubectl, and disables swap.

---

## 📦 Usage

### 1. **Configure Proxmox Host Inventory**
Edit `inventory/proxmox_hosts.ini`

### 2. **Set Proxmox Variables**
Edit `group_vars/proxmox.yml` as needed.

### 3. **Create Ubuntu Template VM**
```bash
ansible-playbook -i inventory/proxmox_hosts.ini playbooks/proxmox_ubuntu_template.yml
```

### 4. **Clone VMs for Kubernetes Nodes**
```bash
ansible-playbook -i inventory/proxmox_hosts.ini playbooks/proxmox_clone_vms.yml
```

### 5. **Configure Cloud-Init for Passwordless Sudo**
Edit `cloudinit/user-data.yaml` and ensure it is attached to your template VM.

### 6. **Define Kubernetes VM Inventory**
Edit `inventory/k8s_vms.ini`

### 7. **Configure VMs for Kubernetes**
```bash
ansible-playbook -i inventory/k8s_vms.ini playbooks/k8s_vm_config.yml
```
You can override the Kubernetes version:
```bash
ansible-playbook -i inventory/k8s_vms.ini playbooks/k8s_vm_config.yml -e k8s_version=v1.34
```

---

## 📚 Next Steps

- Initialize your Kubernetes cluster with `kubeadm init` and join nodes.
- Extend playbooks for networking, monitoring, and more.

---

## 🔐 Security

<!-- All sensitive credentials (usernames, passwords) are encrypted using Ansible Vault. See [SECURITY.md](SECURITY.md) for detailed instructions on managing encrypted credentials. -->

## 🧑‍💻 Requirements

- Proxmox VE with SSH access
- Ansible 2.9+
- Cloud-init enabled template
- Passwordless sudo for Ansible user on VMs