# Kubernetes Applications Templates

Templates for components that deploy Kubernetes resources (secrets, deployments, helm charts).

## install.yml Templates

### Basic k8s resource (secret, configmap, namespace)

```yaml
---
- name: Create {resource_name}
  hosts: control

  pre_tasks:
    - name: Load secrets from sops file
      community.sops.load_vars:
        file: "{{ secrets_file }}"
      when: secrets_file is defined

  tasks:
    - name: Ensure namespace exists
      kubernetes.core.k8s:
        kubeconfig: "{{ k8s_kubeconfig }}"
        validate_certs: false
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ k8s_namespace }}"
      tags:
        - namespace

    - name: Create {resource_type}
      kubernetes.core.k8s:
        kubeconfig: "{{ k8s_kubeconfig }}"
        validate_certs: false
        state: present
        namespace: "{{ k8s_namespace }}"
        definition:
          apiVersion: v1
          kind: Secret
          metadata:
            name: "{{ secret_name }}"
          type: kubernetes.io/dockerconfigjson
          data:
            .dockerconfigjson: "{{ dockerconfigjson | b64encode }}"
      tags:
        - {tag_name}
```

### Helm chart deployment

```yaml
---
- name: Install {component_name}
  hosts: control

  pre_tasks:
    - name: Load secrets from sops file
      community.sops.load_vars:
        file: "{{ secrets_file }}"
      when: secrets_file is defined

  tasks:
    - name: Deploy {release_name}
      kubernetes.core.helm:
        kubeconfig: "{{ kubeconfig }}"
        validate_certs: false
        release_namespace: "{{ namespace }}"
        create_namespace: true
        name: "{{ release_name }}"
        chart_ref: "{{ chart }}"
        values: "{{ lookup('template', playbook_dir + '/templates/values.yaml') | from_yaml }}"
```

### Multiple k8s resources with templates

```yaml
---
- name: Setup {component_name}
  hosts: control

  pre_tasks:
    - name: Load secrets from sops file
      community.sops.load_vars:
        file: "{{ secrets_file }}"
      when: secrets_file is defined

  tasks:
    - name: Create secret from template
      kubernetes.core.k8s:
        kubeconfig: "{{ k8s_kubeconfig }}"
        validate_certs: false
        state: present
        namespace: "{{ k8s_namespace }}"
        template: "{{ playbook_dir }}/templates/secret.yaml"
      tags:
        - secrets

    - name: Deploy helm chart
      kubernetes.core.helm:
        kubeconfig: "{{ k8s_kubeconfig }}"
        validate_certs: false
        release_namespace: "{{ k8s_namespace }}"
        name: "{{ release_name }}"
        chart_ref: "{{ chart }}"
        values: "{{ lookup('template', playbook_dir + '/templates/values.yaml') | from_yaml }}"
      tags:
        - helm
```

## inventory.yml Templates

### Single namespace inventory

```yaml
control:
  hosts:
    local:
      ansible_connection: local
  vars:
    repo_root: "{{ lookup('ansible.builtin.env', 'REPO_ROOT') }}"
    env_root: "{{ lookup('ansible.builtin.env', 'ACTIVE_ENV_ROOT') }}"

    secrets_file: "{{ inventory_dir }}/secrets.sops.yaml"

    k8s_kubeconfig: "{{ env_root }}/k8s/kubeconfig.yaml"
    k8s_namespace: "{namespace}"

    # Component-specific variables
    release_name: "{release}"
    chart: "{{ repo_root }}/_charts/{chart_name}"
```

### Multi-namespace inventory structure

```
inventory/
└── real/
    ├── ns-dev/
    │   └── inventory.yml    # k8s_namespace: dev
    ├── ns-prod/
    │   └── inventory.yml    # k8s_namespace: prod
    └── ns-tools/
        └── inventory.yml    # k8s_namespace: tools
```

Each inventory:

```yaml
control:
  hosts:
    local:
      ansible_connection: local
  vars:
    repo_root: "{{ lookup('ansible.builtin.env', 'REPO_ROOT') }}"
    env_root: "{{ lookup('ansible.builtin.env', 'ACTIVE_ENV_ROOT') }}"

    secrets_file: "{{ inventory_dir }}/../secrets.sops.yaml"

    k8s_kubeconfig: "{{ env_root }}/k8s/kubeconfig.yaml"
    k8s_namespace: "{namespace}"

    # Instance-specific variables
```

## templates/values.yaml Template

```yaml
---
# Image configuration
images:
  main: "{registry}/{image}:{tag}"
  imagePullSecrets:
    - name: project-registry

# Resource configuration
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Secrets (from inventory variables)
secret:
  accessKey: "{{ access_key }}"
  secretKey: "{{ secret_key }}"

# Configuration
config:
  endpoint: "{{ endpoint_url }}"
```

## templates/secret.yaml Template

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: "{{ secret_name }}"
  namespace: "{{ k8s_namespace }}"
type: Opaque
stringData:
  {% for key, value in secret_data.items() %}
  {{ key }}: "{{ value }}"
  {% endfor %}
```

## Examples from Repository

### docker-registry-secret

**Purpose:** Create docker registry secret in Kubernetes namespace

**install.yml:** Uses `kubernetes.core.k8s` to create Secret of type `kubernetes.io/dockerconfigjson`

**inventory:** One per namespace (`ns-dev`, `ns-prod`, `ns-tools`, `ns-csi-s3`)

**Key variables:** `registry_secret_name`, `docker_registry_url`, `docker_registry_username`, `docker_registry_password`

### csi-s3

**Purpose:** Deploy S3 CSI driver via Helm

**install.yml:** Uses `kubernetes.core.helm` with values from template

**templates/values.yaml:** Contains image references and S3 credentials

**Key variables:** `kubeconfig`, `namespace`, `release_name`, `chart`, `s3_client_id`, `s3_secret_key`

### k8s-app-namespace

**Purpose:** Create namespace with secrets, PVCs, and deploy nginx/imgproxy

**install.yml:** Multiple tasks - namespace, secrets from templates, PVC, helm releases

**Key patterns:**
- Uses `template` parameter in `kubernetes.core.k8s`
- Loops for creating multiple secrets
- Multiple helm deployments in one playbook
- Tags for selective execution
