# Server Applications Templates

Templates for components that install software on servers via systemd and docker-compose.

## install.yml Templates

### Basic systemd + docker-compose service

```yaml
---
- name: Install {service_name}
  hosts: {host_group}
  become: true

  pre_tasks:
    - name: Load secrets from sops file
      community.sops.load_vars:
        file: "{{ secrets_file }}"
      when: secrets_file is defined

  handlers:
    - name: restart {service_name}
      systemd_service:
        daemon_reload: true
        state: restarted
        name: {service_name}

  tasks:
    - name: Create directories
      file:
        path: "{{ item }}"
        state: directory
        mode: 0755
      loop:
        - "{{ installation_dir }}"
        - "{{ installation_dir }}/data"

    - name: Copy templates
      template:
        src: "templates/{{ item.src }}"
        dest: "{{ installation_dir }}/{{ item.dest }}"
      loop:
        - src: docker-compose.yml
          dest: docker-compose.yml
        - src: {service_name}.service
          dest: {service_name}.service
      notify:
        - restart {service_name}

    - name: Symlink service file
      file:
        src: "{{ installation_dir }}/{service_name}.service"
        dest: /usr/lib/systemd/system/{service_name}.service
        state: link

    - name: Start and enable service
      systemd_service:
        state: started
        enabled: true
        daemon_reload: true
        name: {service_name}
```

### Service with static files

```yaml
---
- name: Install {service_name}
  hosts: {host_group}
  become: true

  pre_tasks:
    - name: Load secrets from sops file
      community.sops.load_vars:
        file: "{{ secrets_file }}"
      when: secrets_file is defined

  handlers:
    - name: restart {service_name}
      systemd_service:
        daemon_reload: true
        state: restarted
        name: {service_name}

  tasks:
    - name: Create directories
      file:
        path: "{{ item }}"
        state: directory
        mode: 0755
      loop:
        - "{{ installation_dir }}"
        - "{{ installation_dir }}/config"
        - "{{ installation_dir }}/secrets"

    - name: Copy static files
      copy:
        src: "files/{{ item }}"
        dest: "{{ installation_dir }}/{{ item }}"
      loop:
        - config/settings.conf
        - scripts/startup.sh
      notify:
        - restart {service_name}

    - name: Copy templates
      template:
        src: "templates/{{ item.src }}"
        dest: "{{ installation_dir }}/{{ item.dest }}"
      loop:
        - src: docker-compose.yml
          dest: docker-compose.yml
        - src: {service_name}.service
          dest: {service_name}.service
      notify:
        - restart {service_name}

    - name: Symlink service file
      file:
        src: "{{ installation_dir }}/{service_name}.service"
        dest: /usr/lib/systemd/system/{service_name}.service
        state: link

    - name: Start and enable service
      systemd_service:
        state: started
        enabled: true
        daemon_reload: true
        name: {service_name}
```

### Service with sops-encrypted files

```yaml
---
- name: Install {service_name}
  hosts: {host_group}
  become: true

  pre_tasks:
    - name: Load secrets from sops file
      community.sops.load_vars:
        file: "{{ secrets_file }}"
      when: secrets_file is defined

  handlers:
    - name: restart {service_name}
      systemd_service:
        daemon_reload: true
        state: restarted
        name: {service_name}

  tasks:
    - name: Create directories
      file:
        path: "{{ installation_dir }}"
        state: directory
        mode: 0755

    - name: Copy secret files
      copy:
        dest: "{{ installation_dir }}/secrets/{{ dest_filename }}"
        content: "{{ decrypted_content }}"
        owner: root
        group: root
        mode: '0600'
      vars:
        decrypted_content: "{{ lookup('community.sops.sops', item) }}"
        dest_filename: "{{ item | basename | regex_replace('\\.sops$', '') }}"
      loop:
        - "{{ repo_root }}/_secrets/{secret_file}.sops"
      notify:
        - restart {service_name}

    - name: Copy templates
      template:
        src: "templates/{{ item }}"
        dest: "{{ installation_dir }}/{{ item }}"
      loop:
        - docker-compose.yml
        - {service_name}.service
      notify:
        - restart {service_name}

    - name: Symlink service file
      file:
        src: "{{ installation_dir }}/{service_name}.service"
        dest: /usr/lib/systemd/system/{service_name}.service
        state: link

    - name: Start and enable service
      systemd_service:
        state: started
        enabled: true
        daemon_reload: true
        name: {service_name}
```

## inventory.yml Templates

### Basic server inventory with bastion

```yaml
{host_group}:
  hosts:
    {host_name}:
      ansible_host: "{private_ip}"
      ansible_user: ec2-user
      ansible_ssh_common_args: '-o ProxyJump=ec2-user@{bastion_ip}'
  vars:
    repo_root: "{{ lookup('ansible.builtin.env', 'REPO_ROOT') }}"
    env_root: "{{ lookup('ansible.builtin.env', 'ACTIVE_ENV_ROOT') }}"
    
    secrets_file: "{{ inventory_dir }}/secrets.sops.yaml"

    installation_dir: /opt/{service_name}
```

### LXC container inventory

```yaml
{host_group}:
  hosts:
    {host_name}:
      ansible_host: "{ip}"
      ansible_user: root
  vars:
    repo_root: "{{ lookup('ansible.builtin.env', 'REPO_ROOT') }}"
    env_root: "{{ lookup('ansible.builtin.env', 'ACTIVE_ENV_ROOT') }}"
    
    secrets_file: "{{ inventory_dir }}/secrets.sops.yaml"

    installation_dir: /opt/{service_name}
```

### Local connection inventory

```yaml
{host_group}:
  hosts:
    localhost:
      ansible_connection: local
  vars:
    repo_root: "{{ lookup('ansible.builtin.env', 'REPO_ROOT') }}"
    env_root: "{{ lookup('ansible.builtin.env', 'ACTIVE_ENV_ROOT') }}"
    
    secrets_file: "{{ inventory_dir }}/secrets.sops.yaml"

    installation_dir: /opt/{service_name}
```

## templates/docker-compose.yml Template

```yaml
services:
  {service_name}:
    image: {registry}/{image}:{tag}
    ports:
      - "{host_port}:{container_port}"
    volumes:
      - "{{ installation_dir }}/data:/data:rw"
    environment:
      - "VAR1={{ var1 }}"
      - "VAR2={{ var2 }}"
    restart: unless-stopped

networks:
  docker_{port}:
    driver: bridge
    driver_opts:
      com.docker.network.enable_ipv6: "false"
      com.docker.network.bridge.name: "docker_{port}"
```

## templates/{service}.service Template

```ini
[Unit]
Description={Service Description}
After=docker.service
Requires=docker.service

[Service]
Type=simple
WorkingDirectory={{ installation_dir }}
ExecStart=/usr/bin/docker compose up
ExecStop=/usr/bin/docker compose down
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## Additional Playbook Templates

### copy-configs.yml

```yaml
---
- hosts: {host_group}
  become: true

  tasks:
    - name: Copy config files
      copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
        owner: root
        group: root
        mode: '0644'
      loop: "{{ config_files }}"

    - name: Validate config
      command: "{validation_command}"
      
    - name: Reload service
      systemd_service:
        name: {service_name}
        state: reloaded
```

### copy-cert.yml

```yaml
---
- hosts: {host_group}
  become: true

  tasks:
    - file:
        path: "/etc/ssl/certs/{{ cert_name }}"
        state: directory
        recurse: yes
        owner: root
        group: root
        mode: '0700'

    - copy:
        dest: "/etc/ssl/certs/{{ cert_name }}/{{ dest_filename }}"
        content: "{{ decrypted_content }}"
        owner: root
        group: root
        mode: '0600'
      vars:
        decrypted_content: "{{ lookup('community.sops.sops', item) }}"
        dest_filename: "{{ item | basename | regex_replace('\\.sops$', '') }}"
      loop:
        - "{{ repo_root }}/_certs/{{ cert_name }}/fullchain.pem.sops"
        - "{{ repo_root }}/_certs/{{ cert_name }}/privkey.pem.sops"
```

## Examples from Repository

### elasticsearch (runner)

**Purpose:** Run Elasticsearch in docker-compose with systemd

**install.yml:**
- Creates installation and data directories
- Copies docker-compose.yml and service file from templates
- Creates symlink to systemd
- Starts and enables service

**inventory:**
- `hosts: elasticsearch-testing`
- SSH via bastion
- Variables: `installation_dir`

**templates/docker-compose.yml:** Elasticsearch image with ports, volumes, ulimits, environment

### jenkins (runner)

**Purpose:** Run Jenkins in docker-compose with custom Dockerfiles

**install.yml:**
- Multiple directory creation
- Static files copy (Dockerfiles, plugins.txt)
- Secret files copy from sops
- Template files (casc.yaml, docker-compose.yml, service)
- Handlers for restart

**inventory:**
- `hosts: docker`
- SSH via bastion
- Variables: `installation_dir`, `jenkins_port`, `jenkins_ingress_domain`

### docker (runner)

**Purpose:** Install and configure Docker on server

**install.yml:** Simpler playbook for docker installation

**inventory:** Multiple environments (`real`, `lxc`)
