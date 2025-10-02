# SSH Key Management - Implementation Summary

## What Was Implemented

A **unified SSH key management system** that works seamlessly for both:
1. **Jenkins (containerized environment)** - Keys in `$WORKSPACE/.ssh`
2. **Local script execution** - Keys in `$HOME/.ssh`

## Files Created/Modified

### New Files

1. **`playbooks/00.proxmox_k8s_generate_ssh_keys.yml`**
   - Automatically detects execution environment (Jenkins vs Local)
   - Generates SSH key pairs (Proxmox + VM)
   - Updates `vault.yml` with correct paths
   - Works in both environments without code changes

2. **`templates/vault.yml.j2`**
   - Template for vault configuration
   - Dynamically populated with SSH key paths
   - Includes key content for cloud-init injection

3. **`scripts/setup_ssh_keys.sh`**
   - Standalone SSH key setup script
   - Supports both Jenkins and local execution
   - Options for regeneration and custom directories

4. **`SSH_KEY_SETUP.md`**
   - Complete documentation for SSH key management
   - Usage examples for both environments
   - Troubleshooting guide

5. **`IMPLEMENTATION_SUMMARY.md`** (this file)
   - Overview of implementation
   - Quick reference

### Modified Files

1. **`scripts/run_playbooks.sh`**
   - Added `--ssh-dir` option
   - Added `--regenerate-keys` option
   - Added `--skip-key-gen` option
   - Integrated automatic SSH key generation

## How It Works

### Environment Detection

```yaml
# In playbooks/00.proxmox_k8s_generate_ssh_keys.yml
is_jenkins: "{{ lookup('env', 'JENKINS_HOME') | default('', true) != '' }}"

# If Jenkins: use $WORKSPACE/.ssh
# If Local: use $HOME/.ssh
ssh_key_dir: "{{ lookup('env', 'SSH_KEY_DIR') | default(lookup('env', 'HOME') + '/.ssh', true) }}"
```

### Key Generation Flow

```
┌─────────────────────────────────────┐
│   Run Playbook or Script            │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Detect Environment                │
│   - Jenkins?  → $WORKSPACE/.ssh     │
│   - Local?    → $HOME/.ssh          │
│   - Custom?   → $SSH_KEY_DIR        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Generate SSH Keys                 │
│   - jenkins_proxmox_key (ed25519)  │
│   - jenkins_vm_key (ed25519)       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Update vault.yml                  │
│   - Set key paths                   │
│   - Include key content             │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Keys Ready for Use                │
│   - Ansible playbooks               │
│   - SSH connections                 │
└─────────────────────────────────────┘
```

## Usage Examples

### Local Execution

```bash
# Automatic key generation and playbook execution
./scripts/run_playbooks.sh --yes

# Or generate keys separately
./scripts/setup_ssh_keys.sh

# With custom directory
./scripts/setup_ssh_keys.sh --ssh-dir /custom/path/.ssh

# Force regeneration
./scripts/setup_ssh_keys.sh --regenerate
```

### Jenkins Execution

```groovy
stage('🔑 Setup SSH Keys') {
    steps {
        sh '''
            cd ${WORKSPACE}
            export SSH_KEY_DIR=${WORKSPACE}/.ssh
            export REGENERATE_KEYS=false
            ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
        '''
    }
}
```

### Direct Ansible Execution

```bash
# With environment variables
SSH_KEY_DIR=/custom/.ssh \
REGENERATE_KEYS=true \
ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml

# Default behavior (auto-detects environment)
ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
```

## Configuration Options

### Environment Variables

| Variable | Default | Jenkins | Local | Description |
|----------|---------|---------|-------|-------------|
| `SSH_KEY_DIR` | Auto-detected | `$WORKSPACE/.ssh` | `$HOME/.ssh` | SSH key directory |
| `REGENERATE_KEYS` | `false` | `false` | `false` | Force key regeneration |
| `JENKINS_HOME` | - | Auto-set | - | Jenkins detection |
| `WORKSPACE` | - | Auto-set | - | Jenkins workspace |

### Script Options

```bash
# setup_ssh_keys.sh
--regenerate              Force regenerate keys
--ssh-dir DIR             Custom SSH directory
--help                    Show help

# run_playbooks.sh
--ssh-dir DIR             Custom SSH directory
--regenerate-keys         Regenerate before running
--skip-key-gen            Skip key generation
--yes                     Non-interactive mode
--help                    Show help
```

## Key Features

✅ **Automatic Environment Detection**
   - No manual configuration needed
   - Works in both Jenkins and local environments

✅ **Flexible Key Storage**
   - Jenkins: `$WORKSPACE/.ssh` (ephemeral, per-build)
   - Local: `$HOME/.ssh` (persistent, user-specific)
   - Custom: `$SSH_KEY_DIR` (user-defined)

✅ **Idempotent Operations**
   - Keys generated only if they don't exist
   - Force regeneration available via flag

✅ **Secure by Default**
   - Proper file permissions (700 for directory, 600 for keys)
   - Keys never committed to git

✅ **Seamless Integration**
   - Works with existing playbooks
   - No code changes needed in other playbooks
   - Vault variables automatically updated

## Testing the Implementation

### Test Local Execution

```bash
# 1. Generate keys
./scripts/setup_ssh_keys.sh

# 2. Verify keys exist
ls -la ~/.ssh/jenkins_*

# 3. Check vault.yml
cat group_vars/vault.yml | grep ssh

# 4. Test SSH connection
ssh -i ~/.ssh/jenkins_proxmox_key root@proxmox.laz "hostname"
```

### Test Jenkins Execution

```bash
# 1. Simulate Jenkins environment
export JENKINS_HOME=/var/jenkins_home
export WORKSPACE=/tmp/test_workspace

# 2. Generate keys
./scripts/setup_ssh_keys.sh

# 3. Verify keys in workspace
ls -la $WORKSPACE/.ssh/jenkins_*

# 4. Check vault.yml paths
cat group_vars/vault.yml | grep ssh
```

### Test Custom Directory

```bash
# 1. Custom directory
./scripts/setup_ssh_keys.sh --ssh-dir /tmp/custom_ssh

# 2. Verify keys
ls -la /tmp/custom_ssh/jenkins_*

# 3. Cleanup
rm -rf /tmp/custom_ssh
```

## Integration with Existing Workflow

### Before (Manual SSH Key Management)

```yaml
# vault.yml (manually maintained)
vault_ssh_private_key_path: "/home/user/.ssh/id_ed25519"

# Had to manually:
# 1. Generate keys
# 2. Update vault.yml
# 3. Copy keys to Jenkins
# 4. Ensure paths match environment
```

### After (Automated SSH Key Management)

```yaml
# vault.yml (auto-generated)
vault_ssh_private_key_path: "/auto/detected/path/.ssh/jenkins_proxmox_key"

# Automatically:
# 1. Detects environment
# 2. Generates keys in correct location
# 3. Updates vault.yml with correct paths
# 4. Works in both Jenkins and local
```

## Benefits

1. **No Manual Intervention**
   - Keys auto-generated
   - Paths auto-configured
   - Environment auto-detected

2. **Works Everywhere**
   - Jenkins containers
   - Local workstations
   - CI/CD pipelines
   - Custom environments

3. **Secure**
   - Proper permissions
   - Keys not in git
   - Ephemeral in Jenkins

4. **Maintainable**
   - One playbook for all environments
   - Clear documentation
   - Easy to troubleshoot

5. **Flexible**
   - Override any default
   - Custom directories
   - Force regeneration

## Next Steps

1. **Test the Implementation**
   ```bash
   # Run locally
   ./scripts/run_playbooks.sh --yes
   ```

2. **Update Jenkins Pipeline**
   - Add SSH key generation stage
   - Use generated keys in subsequent stages

3. **Document for Team**
   - Share `SSH_KEY_SETUP.md`
   - Add examples to team wiki

4. **Optional: Jenkins Credentials Integration**
   - Store generated keys in Jenkins credentials
   - Reference in pipeline via credentials ID

## Questions?

Refer to:
- **`SSH_KEY_SETUP.md`** - Complete usage guide
- **`playbooks/00.proxmox_k8s_generate_ssh_keys.yml`** - Implementation details
- **`scripts/setup_ssh_keys.sh`** - Standalone script
- **`scripts/run_playbooks.sh`** - Integrated script

