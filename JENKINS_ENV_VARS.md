# Jenkins Environment Variables Guide

## Yes! Jenkins Automatically Defines These Variables

When Jenkins executes a pipeline, it automatically provides several environment variables, including:

### Core Jenkins Environment Variables

| Variable | Always Available? | Example Value | Description |
|----------|------------------|---------------|-------------|
| `JENKINS_HOME` | ✅ Yes | `/var/jenkins_home` | Jenkins installation directory |
| `WORKSPACE` | ✅ Yes | `/var/jenkins_home/workspace/my-job` | Current job workspace |
| `BUILD_NUMBER` | ✅ Yes | `42` | Current build number |
| `BUILD_ID` | ✅ Yes | `42` | Current build ID |
| `BUILD_URL` | ✅ Yes | `http://jenkins/job/my-job/42/` | URL to build |
| `JOB_NAME` | ✅ Yes | `my-job` | Name of the job |
| `NODE_NAME` | ✅ Yes | `master` or agent name | Node executing the build |

## How Our SSH Key Solution Uses These

### In playbooks/00.proxmox_k8s_generate_ssh_keys.yml

```yaml
vars:
  # Detects if running in Jenkins
  is_jenkins: "{{ lookup('env', 'JENKINS_HOME') | default('', true) != '' }}"
  
  # Gets workspace directory
  workspace_dir: "{{ lookup('env', 'WORKSPACE') | default(playbook_dir + '/..', true) }}"
  
  # Determines SSH key directory based on environment
  ssh_key_dir: "{{ workspace_dir }}/.ssh" if is_jenkins else "$HOME/.ssh"
```

### Detection Logic

```yaml
# If JENKINS_HOME is set (non-empty):
#   → is_jenkins = true
#   → ssh_key_dir = $WORKSPACE/.ssh
#
# If JENKINS_HOME is NOT set:
#   → is_jenkins = false
#   → ssh_key_dir = $HOME/.ssh
```

## Testing Jenkins Environment Detection

### 1. In Your Jenkinsfile

You can verify these variables are available:

```groovy
stage('🔍 Verify Jenkins Environment') {
    steps {
        sh '''
            echo "JENKINS_HOME: ${JENKINS_HOME}"
            echo "WORKSPACE: ${WORKSPACE}"
            echo "BUILD_NUMBER: ${BUILD_NUMBER}"
            echo "JOB_NAME: ${JOB_NAME}"
            echo "NODE_NAME: ${NODE_NAME}"
            
            # Check if variables are set
            if [ -n "${JENKINS_HOME}" ]; then
                echo "✅ JENKINS_HOME is set"
            else
                echo "❌ JENKINS_HOME is NOT set"
            fi
            
            if [ -n "${WORKSPACE}" ]; then
                echo "✅ WORKSPACE is set: ${WORKSPACE}"
            else
                echo "❌ WORKSPACE is NOT set"
            fi
        '''
    }
}
```

### 2. Add to Your Existing Jenkinsfile

You can add this to your Pre-flight Checks stage:

```groovy
stage('🔍 Pre-flight Checks') {
    steps {
        script {
            echo "🚀 Starting Kubernetes Cluster Deployment"
            echo "Full Cluster Mode: ${params.FULL_CLUSTER}"
            echo "Ansible Verbosity: ${params.ANSIBLE_VERBOSITY}"
            
            // Display Jenkins environment
            echo "📊 Jenkins Environment:"
            echo "  JENKINS_HOME: ${env.JENKINS_HOME}"
            echo "  WORKSPACE: ${env.WORKSPACE}"
            echo "  BUILD_NUMBER: ${env.BUILD_NUMBER}"
            echo "  JOB_NAME: ${env.JOB_NAME}"
            
            // Check workspace and inventory files
            sh '''
                echo "📂 Checking workspace structure..."
                echo "  Current directory: $(pwd)"
                echo "  WORKSPACE env var: ${WORKSPACE}"
                ls -la ${WORKSPACE_DIR}/
                
                if [ ! -d "${WORKSPACE_DIR}/playbooks" ]; then
                    echo "❌ Playbooks directory not found"
                    exit 1
                fi
                
                echo "✅ Workspace structure validated"
            '''
        }
    }
}
```

## Your Current Implementation

### Jenkinsfile (Already Using WORKSPACE)

```groovy
environment {
    ANSIBLE_HOST_KEY_CHECKING = 'False'
    ANSIBLE_STDOUT_CALLBACK = 'yaml'
    WORKSPACE_DIR = "${WORKSPACE}"  # ✅ Using Jenkins WORKSPACE
    K8S_INVENTORY = "${WORKSPACE_DIR}/inventory/k8s_vms.ini"
    PROXMOX_INVENTORY = "${WORKSPACE_DIR}/inventory/proxmox.ini"
    VAULT_FILE = "${WORKSPACE_DIR}/group_vars/vault.yml"
}
```

**This is CORRECT!** ✅ Jenkins will automatically substitute `${WORKSPACE}`.

### Example Values in Jenkins

When Jenkins runs your pipeline:

```bash
JENKINS_HOME=/var/jenkins_home
WORKSPACE=/var/jenkins_home/workspace/proxmox-k8s-orchestrator
WORKSPACE_DIR=/var/jenkins_home/workspace/proxmox-k8s-orchestrator
K8S_INVENTORY=/var/jenkins_home/workspace/proxmox-k8s-orchestrator/inventory/k8s_vms.ini
```

## Adding SSH Key Generation to Your Pipeline

### Option 1: Add Stage Before Infrastructure Setup

```groovy
stage('🔑 Generate SSH Keys') {
    steps {
        script {
            echo "🔑 Generating SSH keys for infrastructure access..."
            sh '''
                cd ${WORKSPACE_DIR}
                
                # Export environment variables for Ansible
                export SSH_KEY_DIR=${WORKSPACE_DIR}/.ssh
                export REGENERATE_KEYS=false
                
                # Run SSH key generation playbook
                ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
                
                # Verify keys were generated
                ls -la ${SSH_KEY_DIR}/
                
                # Display key fingerprint
                ssh-keygen -lf ${SSH_KEY_DIR}/jenkins_infra_key.pub
            '''
        }
    }
}
```

### Option 2: Integrate with Infrastructure Setup

```groovy
stage('🏗️ Infrastructure Setup') {
    steps {
        echo "🏗️ Setting up infrastructure..."
        sh """
            cd ${WORKSPACE_DIR}
            
            # Generate SSH keys first
            echo "🔑 Generating SSH keys..."
            export SSH_KEY_DIR=${WORKSPACE_DIR}/.ssh
            export REGENERATE_KEYS=false
            ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
            
            # Generate configurations
            echo "📋 Generating inventory and configurations..."
            ansible-playbook ${params.ANSIBLE_VERBOSITY} \
                playbooks/01.proxmox_k8s_generate_configs.yml \
                -e "include_workers=${params.FULL_CLUSTER}"
            
            # ... rest of infrastructure setup
        """
    }
}
```

## Environment Variable Availability

### In Shell Blocks

```groovy
sh '''
    # These are available:
    echo ${JENKINS_HOME}     # ✅ Available
    echo ${WORKSPACE}        # ✅ Available
    echo ${BUILD_NUMBER}     # ✅ Available
'''
```

### In Groovy Scripts

```groovy
script {
    // Access via env object
    echo "JENKINS_HOME: ${env.JENKINS_HOME}"   # ✅ Available
    echo "WORKSPACE: ${env.WORKSPACE}"         # ✅ Available
    
    // Or directly (in environment block)
    echo "WORKSPACE_DIR: ${WORKSPACE_DIR}"     # ✅ Available (from environment block)
}
```

### In Ansible Playbooks

```yaml
tasks:
  - name: Check if running in Jenkins
    debug:
      msg: "Running in Jenkins: {{ lookup('env', 'JENKINS_HOME') != '' }}"
      
  - name: Display workspace
    debug:
      msg: "Workspace: {{ lookup('env', 'WORKSPACE') | default('Not in Jenkins', true) }}"
```

## Troubleshooting

### Variable Not Set?

If `JENKINS_HOME` or `WORKSPACE` appear empty:

```groovy
stage('Debug Environment') {
    steps {
        sh '''
            echo "=== All Environment Variables ==="
            env | grep -i jenkins
            env | grep -i workspace
            
            echo "=== Current Directory ==="
            pwd
            
            echo "=== Directory Contents ==="
            ls -la
        '''
    }
}
```

### Docker Agent Issue

If using Docker agents, ensure variables are passed:

```groovy
agent {
    docker {
        image 'ansible:latest'
        args '-e JENKINS_HOME -e WORKSPACE'  // ✅ Pass variables
    }
}
```

## Summary

### ✅ Yes, These Are Always Defined in Jenkins

```bash
JENKINS_HOME  # Always set (e.g., /var/jenkins_home)
WORKSPACE     # Always set (e.g., /var/jenkins_home/workspace/job-name)
```

### ✅ Your Implementation is Correct

```groovy
environment {
    WORKSPACE_DIR = "${WORKSPACE}"  # ✅ This works!
}
```

### ✅ SSH Key Playbook Will Detect Jenkins

```yaml
is_jenkins: "{{ lookup('env', 'JENKINS_HOME') | default('', true) != '' }}"
# ✅ This will be TRUE in Jenkins, FALSE locally
```

### ✅ Keys Will Be Stored in Correct Location

```
Jenkins:  $WORKSPACE/.ssh/jenkins_infra_key
Local:    $HOME/.ssh/jenkins_infra_key
```

## Complete Example Pipeline with SSH Keys

```groovy
pipeline {
    agent any
    
    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        WORKSPACE_DIR = "${WORKSPACE}"
        SSH_KEY_DIR = "${WORKSPACE}/.ssh"
        K8S_INVENTORY = "${WORKSPACE_DIR}/inventory/k8s_vms.ini"
        PROXMOX_INVENTORY = "${WORKSPACE_DIR}/inventory/proxmox.ini"
    }
    
    stages {
        stage('🔍 Environment Info') {
            steps {
                sh '''
                    echo "📊 Jenkins Environment:"
                    echo "  JENKINS_HOME: ${JENKINS_HOME}"
                    echo "  WORKSPACE: ${WORKSPACE}"
                    echo "  SSH_KEY_DIR: ${SSH_KEY_DIR}"
                    echo "  BUILD_NUMBER: ${BUILD_NUMBER}"
                '''
            }
        }
        
        stage('🔑 Generate SSH Keys') {
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    export REGENERATE_KEYS=false
                    ansible-playbook playbooks/00.proxmox_k8s_generate_ssh_keys.yml
                    
                    # Verify
                    ls -la ${SSH_KEY_DIR}/
                    ssh-keygen -lf ${SSH_KEY_DIR}/jenkins_infra_key.pub
                '''
            }
        }
        
        stage('🏗️ Infrastructure Setup') {
            steps {
                sh '''
                    cd ${WORKSPACE_DIR}
                    ansible-playbook -i ${PROXMOX_INVENTORY} \
                        playbooks/02.proxmox_k8s_create_vm_template.yml
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "🧹 Cleaning up SSH keys from workspace..."
                rm -rf ${SSH_KEY_DIR}
            '''
        }
    }
}
```

**Everything works automatically - Jenkins provides the variables, Ansible detects them!** ✅

