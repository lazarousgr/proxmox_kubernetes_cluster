# SSH Key Approach Comparison

## Visual Comparison

### ❌ Old Approach: Two Separate Key Pairs

```
┌─────────────────────────────────────────────────────────────────┐
│                  LOCAL WORKSTATION / JENKINS                     │
│                                                                  │
│  ~/.ssh/ or $WORKSPACE/.ssh/                                    │
│  ├── jenkins_proxmox_key      ◄─── For Proxmox only            │
│  ├── jenkins_proxmox_key.pub                                   │
│  ├── jenkins_vm_key           ◄─── For VMs only                │
│  └── jenkins_vm_key.pub                                         │
│                                                                  │
└────────────────┬──────────────────────┬─────────────────────────┘
                 │                      │
                 │ jenkins_proxmox_key  │ jenkins_vm_key
                 │                      │
         ┌───────▼────────┐    ┌────────▼────────┐
         │  Proxmox Host  │    │   K8s VMs       │
         │                │    │  - Master       │
         │  root@proxmox  │    │  - Worker 1     │
         │                │    │  - Worker 2     │
         └────────────────┘    └─────────────────┘

Issues:
  ❌ Two keys to generate
  ❌ Two keys to distribute
  ❌ Two keys to manage
  ❌ Two keys to rotate
  ❌ Confusion about which key for what
```

### ✅ New Approach: Single Key Pair

```
┌─────────────────────────────────────────────────────────────────┐
│                  LOCAL WORKSTATION / JENKINS                     │
│                                                                  │
│  ~/.ssh/ or $WORKSPACE/.ssh/                                    │
│  ├── jenkins_infra_key        ◄─── For EVERYTHING              │
│  └── jenkins_infra_key.pub                                      │
│                                                                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             │ jenkins_infra_key (single key)
                             │
         ┌───────────────────┴─────────────────────┐
         │                                         │
         ▼                                         ▼
┌────────────────┐                        ┌─────────────────┐
│  Proxmox Host  │                        │   K8s VMs       │
│                │                        │  - Master       │
│  root@proxmox  │                        │  - Worker 1     │
│                │                        │  - Worker 2     │
└────────────────┘                        └─────────────────┘

Benefits:
  ✅ One key to generate
  ✅ One key to distribute
  ✅ One key to manage
  ✅ One key to rotate
  ✅ Clear and simple
```

## Authentication Flow Comparison

### Old Approach: Two Keys

```
                     Proxmox Access
┌──────────┐                              ┌──────────┐
│ Workst/  │  ssh -i jenkins_proxmox_key  │ Proxmox  │
│ Jenkins  │ ────────────────────────────►│   Host   │
└──────────┘                              └──────────┘
     │
     │ Uses: jenkins_proxmox_key
     └─────────────────────────────────────────────────┐
                                                        │
                     VM Access                         │
┌──────────┐                              ┌──────────┐ │
│ Workst/  │  ssh -i jenkins_vm_key       │   VMs    │ │
│ Jenkins  │ ────────────────────────────►│ (nodes)  │ │
└──────────┘                              └──────────┘ │
     │                                                  │
     │ Uses: jenkins_vm_key                            │
     └─────────────────────────────────────────────────┘
                                                        
          ⚠️  Two different keys to track
```

### New Approach: Single Key

```
                  All Infrastructure Access
┌──────────┐                              ┌──────────┐
│ Workst/  │                              │ Proxmox  │
│ Jenkins  │  ssh -i jenkins_infra_key    │   Host   │
│          │ ────────────────────────────►│          │
│          │                              └──────────┘
│          │                                    
│          │                              ┌──────────┐
│          │  ssh -i jenkins_infra_key    │   VMs    │
│          │ ────────────────────────────►│ (nodes)  │
└──────────┘                              └──────────┘

     ✅ Single key for everything
```

## Vault Configuration Comparison

### Old Approach: vault.yml (Two Keys)

```yaml
# Proxmox access
vault_ssh_private_key_path: "/path/.ssh/jenkins_proxmox_key"
vault_ssh_public_key_path: "/path/.ssh/jenkins_proxmox_key.pub"
vault_proxmox_ssh_public_key: "ssh-ed25519 AAAA... proxmox-key"

# VM access
vault_ci_ssh_private_key_path: "/path/.ssh/jenkins_vm_key"
vault_ci_ssh_public_key_path: "/path/.ssh/jenkins_vm_key.pub"
vault_ci_ssh_public_key: "ssh-ed25519 AAAA... vm-key"

# Problem: Two sets of paths, confusion about which to use
```

### New Approach: vault.yml (Single Key)

```yaml
# Single SSH key configuration
vault_ssh_private_key_path: "/path/.ssh/jenkins_infra_key"
vault_ssh_public_key_path: "/path/.ssh/jenkins_infra_key.pub"
vault_ssh_public_key: "ssh-ed25519 AAAA... infra-key"

# Proxmox uses vault_ssh_private_key_path
vault_proxmox_user: "root"

# VMs also use vault_ssh_private_key_path
vault_ci_user: "lazarous"
vault_ci_ssh_private_key_path: "{{ vault_ssh_private_key_path }}"
vault_ci_ssh_public_key_path: "{{ vault_ssh_public_key_path }}"

# Solution: Single source of truth
```

## Key Management Workflow

### Old Approach

```
Step 1: Generate Keys
├── Generate jenkins_proxmox_key
└── Generate jenkins_vm_key

Step 2: Distribute Keys
├── Copy jenkins_proxmox_key.pub to Proxmox
└── Inject jenkins_vm_key.pub into VM cloud-init

Step 3: Manage Keys
├── Track jenkins_proxmox_key location
└── Track jenkins_vm_key location

Step 4: Rotate Keys (when needed)
├── Regenerate jenkins_proxmox_key
├── Redistribute to Proxmox
├── Regenerate jenkins_vm_key
└── Redistribute to VMs

⏱️  Time: ~10 minutes
🔧 Complexity: Medium-High
```

### New Approach

```
Step 1: Generate Key
└── Generate jenkins_infra_key

Step 2: Distribute Key
├── Copy jenkins_infra_key.pub to Proxmox
└── Inject jenkins_infra_key.pub into VM cloud-init

Step 3: Manage Key
└── Track jenkins_infra_key location

Step 4: Rotate Key (when needed)
├── Regenerate jenkins_infra_key
└── Redistribute to all hosts

⏱️  Time: ~5 minutes
🔧 Complexity: Low
```

## Command Comparison

### Old Approach: Commands

```bash
# Generate keys
ssh-keygen -t ed25519 -f ~/.ssh/jenkins_proxmox_key
ssh-keygen -t ed25519 -f ~/.ssh/jenkins_vm_key

# Distribute keys
ssh-copy-id -i ~/.ssh/jenkins_proxmox_key.pub root@proxmox.laz
# (VM key via cloud-init)

# Use keys
ssh -i ~/.ssh/jenkins_proxmox_key root@proxmox.laz
ssh -i ~/.ssh/jenkins_vm_key lazarous@192.168.1.41

# Rotate keys
ssh-keygen -t ed25519 -f ~/.ssh/jenkins_proxmox_key -N "" -y
ssh-keygen -t ed25519 -f ~/.ssh/jenkins_vm_key -N "" -y
# ... distribute both again

# ❌ Many commands, easy to forget which key for what
```

### New Approach: Commands

```bash
# Generate key (automated)
./scripts/setup_ssh_keys.sh

# Distribute key
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz
# (Same key via cloud-init for VMs)

# Use key
ssh -i ~/.ssh/jenkins_infra_key root@proxmox.laz
ssh -i ~/.ssh/jenkins_infra_key lazarous@192.168.1.41

# Rotate key
./scripts/setup_ssh_keys.sh --regenerate
# ... distribute once

# ✅ Simple, one key for everything
```

## Inventory Configuration

### Old Approach

```ini
# inventory/proxmox.ini
[proxmox]
proxmox.laz ansible_ssh_private_key_file={{ vault_ssh_private_key_path }} 
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            Uses jenkins_proxmox_key

# inventory/k8s_vms.ini
[k8s_vms]
192.168.1.41 ansible_ssh_private_key_file={{ vault_ci_ssh_private_key_path }}
                                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                            Uses jenkins_vm_key

# Problem: Two different variables to remember
```

### New Approach

```ini
# inventory/proxmox.ini
[proxmox]
proxmox.laz ansible_ssh_private_key_file={{ vault_ssh_private_key_path }}
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            Uses jenkins_infra_key

# inventory/k8s_vms.ini
[k8s_vms]
192.168.1.41 ansible_ssh_private_key_file={{ vault_ssh_private_key_path }}
                                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                            Uses jenkins_infra_key (same!)

# Solution: Single variable everywhere
```

## Security Comparison

### Old Approach Security

```
🔐 Proxmox Key:
   ✅ Separate key for Proxmox
   ✅ Root user access controlled
   ⚠️  More keys = more attack surface

🔐 VM Key:
   ✅ Separate key for VMs
   ✅ Non-root user access
   ⚠️  More keys = more to secure

Overall:
   ✅ Separation of concerns
   ❌ More complexity
   ❌ More keys to rotate
   ❌ Higher chance of misconfiguration
```

### New Approach Security

```
🔐 Single Infrastructure Key:
   ✅ One key to secure
   ✅ Different users (root vs lazarous) provide separation
   ✅ Host-based access control
   ✅ ED25519 encryption
   ✅ Easier to rotate
   ✅ Less attack surface (fewer keys)

Overall:
   ✅ Simpler = more secure
   ✅ Easier to audit
   ✅ Less chance of misconfiguration
   ✅ Still maintains separation via users
```

## File Complexity

### Old Approach: File Count

```
Files to manage:
  ~/.ssh/jenkins_proxmox_key           (private)
  ~/.ssh/jenkins_proxmox_key.pub       (public)
  ~/.ssh/jenkins_vm_key                (private)
  ~/.ssh/jenkins_vm_key.pub            (public)
  
  Total: 4 files
  
Vault variables:
  vault_ssh_private_key_path
  vault_ssh_public_key_path
  vault_proxmox_ssh_public_key
  vault_ci_ssh_private_key_path
  vault_ci_ssh_public_key_path
  vault_ci_ssh_public_key
  
  Total: 6 variables
```

### New Approach: File Count

```
Files to manage:
  ~/.ssh/jenkins_infra_key             (private)
  ~/.ssh/jenkins_infra_key.pub         (public)
  
  Total: 2 files (50% reduction)
  
Vault variables:
  vault_ssh_private_key_path
  vault_ssh_public_key_path
  vault_ssh_public_key
  
  Total: 3 variables (50% reduction)
```

## Summary Table

| Aspect | Old (Two Keys) | New (Single Key) | Winner |
|--------|----------------|------------------|--------|
| **Number of Keys** | 2 key pairs (4 files) | 1 key pair (2 files) | ✅ New |
| **Complexity** | Medium-High | Low | ✅ New |
| **Generation Time** | ~2 minutes | ~1 minute | ✅ New |
| **Distribution Effort** | 2 keys to distribute | 1 key to distribute | ✅ New |
| **Vault Variables** | 6 variables | 3 variables | ✅ New |
| **Management Overhead** | High (track 2 keys) | Low (track 1 key) | ✅ New |
| **Rotation Complexity** | Rotate 2 keys | Rotate 1 key | ✅ New |
| **Security** | Good (but complex) | Good (simpler) | ✅ New |
| **User Separation** | Via different keys | Via different users | 🟰 Same |
| **Clarity** | Which key for what? | One key for all | ✅ New |

## Conclusion

### Why Single Key is Better

1. **50% fewer files** to manage
2. **50% fewer variables** in vault
3. **Simpler mental model** - one key does it all
4. **Faster setup** - generate once, use everywhere
5. **Easier rotation** - rotate one key instead of two
6. **Less error-prone** - no confusion about which key
7. **Maintains security** - user separation still applies

### Migration Path

If using old approach:

```bash
# 1. Remove old keys
rm ~/.ssh/jenkins_proxmox_key* ~/.ssh/jenkins_vm_key*

# 2. Generate new single key
./scripts/setup_ssh_keys.sh

# 3. Distribute to Proxmox
ssh-copy-id -i ~/.ssh/jenkins_infra_key.pub root@proxmox.laz

# 4. Redeploy VMs (cloud-init injects new key)
./scripts/run_playbooks.sh --yes

# Done! ✅
```

**Result: Simpler, cleaner, easier! 🎉**

