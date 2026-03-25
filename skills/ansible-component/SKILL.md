---
name: ansible-component
description: Work with DevOps components in IaC repository. Use when user mentions creating, modifying, or analyzing ansible playbooks for DevOps components like kubernetes resources, server applications, configs, secrets, helm charts, systemd services. Triggers on: "create component", "new playbook", "add inventory", "ansible component", "devops component", "helm chart", "k8s secret", "systemd service", "docker-compose service".
---

# Ansible Component Skill

Manage DevOps components in Infrastructure-as-Code repository. Each component is a folder with ansible playbooks and inventories for different environments/instances.

## Component Structure

```
{env}/{category}/{component}/
├── install.yml              # Main installation playbook
├── inventory/
│   └── {environment}/       # real, lxc, etc.
│       ├── inventory.yml    # Single inventory
│       └── {instance}/      # Multiple instances (ns-dev, ns-prod)
│           └── inventory.yml
├── templates/               # Jinja2 templates (optional)
├── files/                   # Static files (optional)
└── roles/                   # Ansible roles (optional)
```

## Operations

### 1. Create New Component

Ask user:
1. **Component name** (e.g., `my-app`, `postgres-backup`)
2. **Category** - existing folder under env (e.g., `kubernetes-apps`, `runner`, `monitoring`)
3. **Component type**: `kubernetes-apps` or `server-apps`
4. **Environment**: `real` (default), `lxc`, etc.

Then:
1. Create component directory structure
2. Generate `install.yml` from appropriate template
3. Generate `inventory/{environment}/inventory.yml`
4. Create necessary template files

### 2. Add Inventory to Existing Component

Ask user:
1. **Component path** (e.g., `dev/kubernetes-apps/my-app`)
2. **New inventory name** (e.g., `ns-staging`, `prod`)
3. **Base inventory to copy from** (optional)

Then:
1. Analyze existing inventories to understand variables
2. Create new inventory directory
3. Copy and modify inventory.yml with new values

### 3. Add Playbook to Existing Component

Ask user:
1. **Component path**
2. **Playbook purpose** (e.g., `copy-configs`, `create-secret`, `backup`)
3. **What the playbook should do**

Then:
1. Analyze existing playbooks and inventories
2. Create new playbook following project patterns
3. Update inventory if new variables needed

### 4. Analyze Existing Components

When user asks to find or analyze components:

1. Search in repository for matching components
2. Read playbooks and inventories (skip files with "sops" in name)
3. Provide summary of:
   - What the component does
   - Available playbooks
   - Available inventories/environments
   - Key variables

## Component Types

### kubernetes-apps

For deploying Kubernetes resources (secrets, deployments, helm charts).

**Key patterns:**
- `hosts: control` with `ansible_connection: local`
- Uses `kubernetes.core.k8s` and `kubernetes.core.helm` modules
- Variables: `kubeconfig`, `namespace`, `repo_root`, `env_root`
- Inventory typically has multiple instances per namespace

**Read templates from:** `references/kubernetes-apps.md`

### server-apps

For installing software on servers via systemd/docker-compose.

**Key patterns:**
- `hosts: <group>` with `become: true`
- SSH connection often via bastion (`ansible_ssh_common_args`)
- Uses `systemd_service`, `template`, `copy`, `file` modules
- Handlers for service restart
- Templates: `docker-compose.yml`, `<service>.service`
- Variable: `installation_dir`

**Read templates from:** `references/server-apps.md`

## Common Patterns

### Pre-tasks for secrets loading

```yaml
pre_tasks:
  - name: Load secrets from sops file
    community.sops.load_vars:
      file: "{{ secrets_file }}"
    when: secrets_file is defined
```

### Standard inventory variables

```yaml
vars:
  repo_root: "{{ lookup('ansible.builtin.env', 'REPO_ROOT') }}"
  env_root: "{{ lookup('ansible.builtin.env', 'ACTIVE_ENV_ROOT') }}"
  secrets_file: "{{ inventory_dir }}/secrets.sops.yaml"
```

### SSH via bastion

```yaml
ansible_ssh_common_args: '-o ProxyJump=ec2-user@<bastion-ip>'
```

## Important Rules

1. **NEVER read or create files with "sops" in filename** - these are encrypted secrets
2. **Always add `secrets_file` variable** to inventory, but don't create the file
3. **Always add pre_tasks** for loading secrets in playbooks
4. **Follow existing patterns** in the repository when creating new components
5. **Use tags** for task groups in complex playbooks

## Workflow

### Creating kubernetes-apps component

1. Read `references/kubernetes-apps.md` for templates
2. Create directory: `{env}/kubernetes-apps/{component}/`
3. Create `install.yml` with k8s/helm tasks
4. Create `inventory/{environment}/{instance}/inventory.yml`
5. Create `templates/` if helm values or k8s manifests needed

### Creating server-apps component

1. Read `references/server-apps.md` for templates
2. Create directory: `{env}/{category}/{component}/`
3. Create `install.yml` with systemd/docker-compose tasks
4. Create `inventory/{environment}/inventory.yml`
5. Create `templates/docker-compose.yml` and `templates/{service}.service`

### Adding inventory

1. Read existing inventory to understand variable structure
2. Create new directory under `inventory/`
3. Copy inventory structure, modify values for new environment
4. Keep `secrets_file` path relative to `inventory_dir`

## File Naming Conventions

- Playbooks: `install.yml`, `copy-configs.yml`, `create-htpasswd.yml`
- Inventory: `inventory.yml`
- Templates: descriptive names like `values.yaml`, `docker-compose.yml`
- Roles: `{role-name}/tasks/main.yml`, `{role-name}/defaults/main.yml`
