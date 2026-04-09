# Learn Ansible (Ops-First, Very Detailed)

Ansible is an **agentless automation tool**: you install it on a single **control node**, then it manages **managed nodes** remotely (commonly over SSH on Linux/Unix). You write **playbooks** that declare the desired state, and Ansible keeps systems in that state with an emphasis on **idempotence** (running the same automation repeatedly should not keep making changes). 

---

## Table of Contents

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

---

## 1) Install Ansible (control node)

The official docs describe installing either:
- `ansible` (full community package, “batteries included”), or
- `ansible-core` (minimal runtime + a smaller set of built-in plugins/modules). 

### Option A (recommended on many systems): install with `pipx`
`pipx` installs Python apps into isolated environments, which helps avoid OS Python packaging restrictions. 

```bash
# Full package
pipx install --include-deps ansible

# Minimal runtime only
pipx install ansible-core
```

Upgrade:
```bash
pipx upgrade --include-injected ansible
```


### Option B: install with `pip`
```bash
python3 -m pip install --user ansible
# or:
python3 -m pip install --user ansible-core
```

Upgrade:
```bash
python3 -m pip install --upgrade --user ansible
```


### Confirm install
```bash
ansible --version
ansible-playbook --version
```

The `ansible` CLI can also show config location and module paths with `--version`. 

---

## 2) Minimal requirements (managed nodes)

Managed nodes typically do **not** need Ansible installed. They generally need:
- **Python** (to run Ansible-generated Python code for most modules)
- an account that can connect via **SSH** with an interactive POSIX shell (for common Linux/Unix automation). 

> Note: there are exceptions (for example, some network modules don’t require Python on the managed device). 

---

## 3) Create a project layout (best practice)

A very common “scales well” layout is: one repo for playbooks/roles, and separate inventories per environment. Ansible’s best-practices documentation includes examples of separating inventories and keeping `group_vars/` and `host_vars/` alongside them. 

Example layout:

```text
ansible-project/
├─ ansible.cfg
├─ inventories/
│  ├─ dev/
│  │  ├─ hosts.ini
│  │  ├─ group_vars/
│  │  │  ├─ all.yml
│  │  │  └─ web.yml
│  │  └─ host_vars/
│  │     └─ web01.yml
│  └─ prod/
│     ├─ hosts.ini
│     ├─ group_vars/
│     └─ host_vars/
├─ playbooks/
│  ├─ site.yml
│  └─ web.yml
└─ roles/
   └─ nginx/
      ├─ tasks/main.yml
      ├─ handlers/main.yml
      ├─ templates/nginx.conf.j2
      └─ defaults/main.yml
```

Why this layout works:
- inventories are isolated (dev/prod drift is explicit)
- variables live near the inventory/playbooks
- roles keep reusable logic tidy 

---

## 4) Inventory (INI/YAML) + inspect inventory

Ansible “composes” inventory from one or more inventory sources (files, plugins, dynamic sources). 

### 4.1 INI inventory example (`inventories/dev/hosts.ini`)
```ini
[web]
web01 ansible_host=203.0.113.10
web02 ansible_host=203.0.113.11

[db]
db01 ansible_host=203.0.113.20

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/dev_key
```

The inventory guide explains how Ansible uses groups/hosts and how it sources variables relative to inventory and playbook directories. 

### 4.2 YAML inventory example (`inventories/dev/hosts.yml`)
```yaml
all:
  vars:
    ansible_user: ubuntu
  children:
    web:
      hosts:
        web01:
          ansible_host: 203.0.113.10
        web02:
          ansible_host: 203.0.113.11
    db:
      hosts:
        db01:
          ansible_host: 203.0.113.20
```

### 4.3 Inspect inventory (must-know command)

`ansible-inventory` shows inventory “as Ansible sees it” and supports `--list`, `--graph`, and per-host output. 

```bash
# Dump full inventory (JSON by default; use -y for YAML)
ansible-inventory -i inventories/dev/hosts.ini --list -y

# Visualize group graph
ansible-inventory -i inventories/dev/hosts.ini --graph

# Include variables in graph output
ansible-inventory -i inventories/dev/hosts.ini --graph --vars

# Show computed vars for a specific host
ansible-inventory -i inventories/dev/hosts.ini --host web01 -y
```


---

## 5) ansible.cfg + config precedence + debugging config

### 5.1 What can configure Ansible?
Ansible supports multiple configuration sources: `ansible.cfg`, environment variables, CLI options, playbook keywords, and variables. The `ansible-config` utility can show settings, defaults, and where a value came from. 

### 5.2 Config precedence (practical takeaway)
The precedence docs emphasize examples like: playbook keywords override configuration settings, and environment variables can override `ansible.cfg`. 

### 5.3 Minimal `ansible.cfg` (project root)
```ini
[defaults]
inventory = inventories/dev/hosts.ini
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

[ssh_connection]
pipelining = True
```

### 5.4 Debug your config (high-value commands)
```bash
# Show effective config and where each value is set
ansible-config dump

# Show all config options
ansible-config list
```


---

## 6) Ad-hoc commands (one-off operations)

The `ansible` CLI runs a **single task** against a set of hosts (pattern), using `-m` for module and `-a` for arguments. 

### 6.1 Connectivity test (ping)
```bash
ansible all -i inventories/dev/hosts.ini -m ping
```

### 6.2 Gather facts (setup)
```bash
ansible web -i inventories/dev/hosts.ini -m ansible.builtin.setup
```
The `setup` module gathers facts about remote hosts. 

### 6.3 Install a package (Ubuntu/Debian) with `apt`
```bash
ansible web -i inventories/dev/hosts.ini -b \
  -m ansible.builtin.apt \
  -a "name=nginx state=present update_cache=true"
```
The `apt` module manages apt packages, and the docs recommend using the FQCN (`ansible.builtin.apt`) to avoid conflicts. 

### 6.4 Copy a file to remote
```bash
ansible web -i inventories/dev/hosts.ini -b \
  -m ansible.builtin.copy \
  -a "src=./files/index.html dest=/var/www/html/index.html mode=0644"
```
The `copy` module copies files/dirs to remote locations and also recommends FQCN usage. 

---

## 7) Your first real playbook (end-to-end example)

Playbooks are automation blueprints in **YAML** that Ansible uses to deploy and configure nodes in an inventory. 

Create: `playbooks/web.yml`

```yaml
---
- name: Configure web servers
  hosts: web
  become: true

  vars:
    nginx_listen_port: 80

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Deploy nginx config from template
      ansible.builtin.template:
        src: ../roles/nginx/templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
        # Validate before overwriting (prevents locking yourself out / breaking config)
        validate: "nginx -t -c %s"
      notify: Restart nginx

    - name: Ensure nginx is enabled and running
      ansible.builtin.service:
        name: nginx
        enabled: true
        state: started

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

Why this is a “good” first playbook:
- uses modules instead of raw shell (more predictable)
- templates config, validates before overwrite
- restarts nginx only when the template changes (handler) 

Run it:
```bash
ansible-playbook -i inventories/dev/hosts.ini playbooks/web.yml
```

---

## 8) Core modules you’ll use daily (apt/copy/template/service)

### 8.1 `apt` (packages on Debian/Ubuntu)
- Designed to manage packages with states like `present/latest` and optional cache updates. 

Example:
```yaml
- name: Install multiple packages
  ansible.builtin.apt:
    name:
      - git
      - curl
      - nginx
    state: present
    update_cache: true
```

### 8.2 `copy` (static files)
- Copies a file or directory structure to the remote host; can set permissions/ownership. 

Example:
```yaml
- name: Copy system banner
  ansible.builtin.copy:
    src: files/motd
    dest: /etc/motd
    owner: root
    group: root
    mode: "0644"
```

### 8.3 `template` (Jinja2 rendered files)
- Templates are processed by **Jinja2**; `src` is on the controller and `dest` is on the managed node. The module supports backups and many file attributes; it also documents special template-related variables like `template_path`, `template_run_date`, etc. 

Example:
```yaml
- name: Render app config
  ansible.builtin.template:
    src: templates/app.conf.j2
    dest: /etc/myapp/app.conf
    mode: "0640"
    backup: true
```

### 8.4 `service` (portable service management)
- The `service` module “controls services on remote hosts” and supports multiple init systems (systemd, SysV, etc.). It acts as a proxy to system-specific modules and may rely on facts (`ansible_service_mgr`) discovered by `setup`.   
- `enabled` controls boot-time enablement; `state` controls running state; `started/stopped` are idempotent; `restarted/reloaded` always act. 

Example:
```yaml
- name: Ensure nginx starts on boot and is running now
  ansible.builtin.service:
    name: nginx
    enabled: true
    state: started
```

---

## 9) Privilege escalation (become)

Ansible’s privilege escalation (“become”) lets tasks run as another user (commonly root) when needed. 

### 9.1 Common patterns
Play-level:
```yaml
- hosts: web
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
```

Task-level:
```yaml
- name: Read protected file
  ansible.builtin.command: cat /etc/shadow
  become: true
```

The become guide covers how `become_user` sets who you become and how different become plugins behave. 

---

## 10) Variables (group_vars/host_vars) + precedence

The variables guide explains **where** variables can be defined and includes a section on **variable precedence** (“where should I put a variable?”). It also notes that some behaviors can be set by variables, config settings, CLI flags, or playbook keywords (e.g., remote user can be set via `ansible_user`, `DEFAULT_REMOTE_USER`, `-u`, or `remote_user`). 

### 10.1 group_vars / host_vars
Example:

`inventories/dev/group_vars/web.yml`
```yaml
nginx_listen_port: 80
```

`inventories/dev/host_vars/web01.yml`
```yaml
nginx_listen_port: 8080
```

### 10.2 Debug “what value is actually used?”
Use:
```bash
ansible-inventory -i inventories/dev/hosts.ini --host web01 -y
```
This shows final composed vars for that host. 

---

## 11) Facts (setup) + custom facts basics

Facts are variables gathered from managed nodes (OS, network, disks, etc.). The docs explain you can set temporary facts with `set_fact` or provide persistent custom facts via the `facts.d` directory, and they are gathered by `setup`. 

### 11.1 Gather only specific facts (filter)
```bash
ansible web -i inventories/dev/hosts.ini \
  -m ansible.builtin.setup -a "filter=ansible_default_ipv4"
```
`setup` supports filtering and subsets. 

---

## 12) Handlers (notify/restart only when needed)

Handlers run only when notified by tasks that changed. If multiple tasks notify the same handler, Ansible runs it once to avoid unnecessary restarts. You can also flush handlers early with `meta: flush_handlers`. 

Pattern:
```yaml
tasks:
  - name: Deploy config
    ansible.builtin.template:
      src: app.conf.j2
      dest: /etc/app.conf
    notify: Restart app

handlers:
  - name: Restart app
    ansible.builtin.service:
      name: myapp
      state: restarted
```


---

## 13) Loops + conditionals (when/until)

### 13.1 Loops (`loop:`)
Loops are documented with common examples like repeating tasks for multiple users/files and using `until` with registered results. 

```yaml
- name: Create users
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
  loop:
    - alice
    - bob
    - charlie
```

### 13.2 Conditionals (`when:`)
The conditionals guide shows `when:` usage and notes you should apply `| bool` for non-boolean strings like “yes/on/1/true” to ensure correct boolean evaluation. 

```yaml
- name: Only run on Ubuntu
  ansible.builtin.debug:
    msg: "Ubuntu host detected"
  when: ansible_facts['distribution'] == "Ubuntu"
```

### 13.3 Retry until success (`until/retries/delay`)
```yaml
- name: Wait for app health endpoint
  ansible.builtin.uri:
    url: "http://localhost:8080/health"
    status_code: 200
  register: health
  until: health.status == 200
  retries: 20
  delay: 3
```
(“until” + register-based polling is a standard Ansible pattern covered in the loops guide.) 

---

## 14) Running playbooks like a pro (tags/limit/check/diff/verbosity)

### 14.1 Must-know `ansible-playbook` options
The `ansible-playbook` CLI supports options like `--tags`, `--skip-tags`, `--list-tasks`, `--list-tags`, `--step`, `--syntax-check`, and `--start-at-task`. 

Examples:
```bash
# Syntax check only
ansible-playbook -i inventories/dev/hosts.ini playbooks/web.yml --syntax-check

# Dry-run + show diffs for templates/small files
ansible-playbook -i inventories/dev/hosts.ini playbooks/web.yml --check --diff

# Run only a subset of hosts
ansible-playbook -i inventories/dev/hosts.ini playbooks/web.yml --limit web01
```
Check/diff mode behavior is documented, including the idea that check mode predicts changes and diff mode shows changes (great together). 

### 14.2 Tags
Tags let you run or skip specific tasks; the tags documentation explains using `--tags` and `--skip-tags` to control execution. 

In playbook:
```yaml
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
  tags: [install]
```

Run:
```bash
ansible-playbook -i inventories/dev/hosts.ini playbooks/web.yml --tags install
ansible-playbook -i inventories/dev/hosts.ini playbooks/web.yml --skip-tags install
```


### 14.3 Verbosity
The `ansible` CLI docs note multiple `-v` levels up to very verbose; `-vvv` is a common starting point and connection debugging can require more. 

---

## 15) Roles + Galaxy + Collections (reusability)

Roles are the primary way to package reusable automation. The roles guide explains that tasks/templates/files within a role can reference role paths without hardcoding absolute paths. 

### 15.1 Create a role skeleton
```bash
ansible-galaxy role init nginx
```

The `ansible-galaxy` CLI supports role and collection management (install, init, etc.). 

### 15.2 Install roles/collections from a `requirements.yml`
The Galaxy user guide documents installing roles (and collections) and the `requirements.yml` format expectations. 

---

## 16) Execution control (strategies/serial/throttle)

Ansible supports different execution strategies (for example, “free” strategy) and controls like `throttle`. The strategies documentation covers these knobs and includes guidance for “run only once” patterns. 

You’ll commonly use:
- `serial:` for rolling updates
- `throttle:` to cap concurrency on a task
- strategy selection for different execution models 

---

## 17) Secrets with Ansible Vault

### 17.1 Encrypt a string (inline secret)
The vault docs describe `ansible-vault encrypt_string` for encrypting a string into a format you can paste into playbooks/vars files. 

```bash
ansible-vault encrypt_string 'SuperSecret123' --name 'db_password'
```

### 17.2 Use vault IDs / password files
The `ansible` and `ansible-inventory` CLIs both document vault-related options like `--vault-id` and `--vault-password-file`. 

---

## 18) Quality: syntax-check, ansible-lint, Molecule (basic touch)

### 18.1 `ansible-playbook --syntax-check`
Supported by the CLI and also used by tooling (like ansible-lint) to fail fast on syntax errors. 

### 18.2 ansible-lint
Ansible Lint is a CLI tool for linting playbooks/roles/collections; docs cover installing via `pip`/`pipx` and usage patterns. 

```bash
pipx install ansible-lint
ansible-lint
```

### 18.3 Molecule (role/playbook testing framework)
Molecule is an Ansible testing framework for developing and testing collections, playbooks, and roles. 

---

## 19) Troubleshooting checklist

When things don’t behave as expected, these commands usually isolate the issue fast:

1) Confirm versions and paths
```bash
ansible --version
```


2) Confirm config precedence / what’s being used
```bash
ansible-config dump
```


3) Confirm inventory is what you think it is
```bash
ansible-inventory -i inventories/dev/hosts.ini --list -y
ansible-inventory -i inventories/dev/hosts.ini --host web01 -y
```


4) Add verbosity
```bash
ansible-playbook -i inventories/dev/hosts.ini playbooks/web.yml -vvv
```
Verbosity levels are documented on the CLI. 

5) Use check/diff to see what would change
```bash
ansible-playbook -i inventories/dev/hosts.ini playbooks/web.yml --check --diff
```
