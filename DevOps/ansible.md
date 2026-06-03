# Ansible — Interview Fundamentals

## What is Ansible?
Ansible is an agentless, idempotent IT automation tool. It uses SSH (or WinRM for Windows) and YAML-based **playbooks** to configure systems, deploy applications, and orchestrate infrastructure. No agent needs to be installed on managed nodes — only Python is required.

---

## Architecture

```
Control Node
  ├── Inventory (hosts/groups)
  ├── Playbooks (YAML)
  ├── Roles (reusable structure)
  └── Modules (unit of work)
        │
        │  SSH / WinRM
        ▼
Managed Nodes (Python required)
```

### Key Components
| Component | Description |
|-----------|-------------|
| **Inventory** | List of managed hosts (static or dynamic) |
| **Playbook** | YAML file defining automation tasks |
| **Task** | Single action (call one module) |
| **Module** | Unit of code that performs work (copy, yum, service...) |
| **Role** | Reusable, structured collection of tasks/vars/templates |
| **Handler** | Task triggered by a `notify` (e.g., restart service) |
| **Fact** | System info gathered at runtime (`ansible_facts`) |
| **Variable** | Dynamic values injected into playbooks |
| **Template** | Jinja2-based config file generation |
| **Vault** | Encrypted storage for secrets |

---

## Inventory

### Static Inventory (`hosts` or `inventory.ini`)
```ini
[webservers]
web01.example.com
web02.example.com ansible_user=ec2-user

[dbservers]
db01.example.com ansible_port=2222

[production:children]
webservers
dbservers

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

### Dynamic Inventory
Script or plugin that queries cloud APIs (AWS, GCP, Azure) to return JSON inventory:
```bash
ansible-inventory -i aws_ec2.yml --list
```

### Useful Commands
```bash
ansible all -i inventory -m ping
ansible webservers -i inventory -m setup    # gather facts
ansible all --list-hosts
```

---

## Playbooks

### Basic Structure
```yaml
---
- name: Deploy web application
  hosts: webservers
  become: true                     # privilege escalation (sudo)
  vars:
    app_version: "2.1.0"
    app_port: 8080

  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present

    - name: Deploy config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart nginx

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted

  post_tasks:
    - name: Verify nginx is responding
      uri:
        url: "http://localhost:{{ app_port }}"
        status_code: 200
```

### Running Playbooks
```bash
ansible-playbook site.yml -i inventory
ansible-playbook site.yml -i inventory --check          # dry-run
ansible-playbook site.yml -i inventory --diff           # show changes
ansible-playbook site.yml -i inventory --limit web01    # single host
ansible-playbook site.yml -i inventory --tags deploy    # run tagged tasks
ansible-playbook site.yml -i inventory --skip-tags test
ansible-playbook site.yml -i inventory -e "version=2.2" # extra vars
ansible-playbook site.yml -i inventory -v / -vvv        # verbosity
```

---

## Core Modules

### Package Management
```yaml
- name: Install packages (yum/apt auto-detected)
  package:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl

- apt:
    name: nginx=1.18.*
    state: present
    update_cache: yes

- yum:
    name: httpd
    state: latest
```

### File Operations
```yaml
- copy:
    src: files/app.conf
    dest: /etc/app/app.conf
    owner: app
    group: app
    mode: '0640'
    backup: yes

- template:
    src: templates/config.j2
    dest: /etc/myapp/config.yml

- file:
    path: /opt/myapp/logs
    state: directory
    mode: '0755'
    owner: appuser

- lineinfile:
    path: /etc/hosts
    regexp: '^127\.0\.0\.1'
    line: '127.0.0.1 localhost'

- blockinfile:
    path: /etc/ssh/sshd_config
    block: |
      MaxSessions 10
      ClientAliveInterval 60
```

### Service Management
```yaml
- service:
    name: nginx
    state: restarted
    enabled: true

- systemd:
    name: myapp
    state: started
    enabled: true
    daemon_reload: true
```

### Command Execution
```yaml
- command: /opt/myapp/bin/migrate --check
  register: migrate_result

- shell: "ps aux | grep myapp | grep -v grep"
  register: proc_check

- raw: apt-get install -y python3  # no Python needed on target

- script: scripts/setup.sh          # copy and run local script
```

### Conditionals & Loops
```yaml
- name: Install only on RedHat
  yum:
    name: httpd
    state: present
  when: ansible_os_family == "RedHat"

- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: alice, groups: sudo }
    - { name: bob, groups: docker }

- name: Retry until service is up
  uri:
    url: http://localhost:8080/health
  register: result
  until: result.status == 200
  retries: 10
  delay: 5
```

---

## Variables

### Precedence (lowest to highest)
1. Role defaults (`defaults/main.yml`)
2. Inventory group vars
3. Inventory host vars
4. Playbook group vars
5. Playbook host vars
6. Task vars
7. Extra vars (`-e` flag) — **highest**

### Variable Files
```yaml
# group_vars/webservers.yml
nginx_worker_processes: 4
app_port: 8080

# host_vars/web01.yml
ansible_user: ec2-user
```

### Registering & Using Output
```yaml
- name: Check if file exists
  stat:
    path: /opt/myapp/INSTALLED
  register: install_flag

- name: Run install only if needed
  include_tasks: install.yml
  when: not install_flag.stat.exists
```

---

## Roles

### Directory Structure
```
roles/
  myapp/
    tasks/
      main.yml
    handlers/
      main.yml
    templates/
      app.conf.j2
    files/
      static_file.conf
    vars/
      main.yml
    defaults/
      main.yml
    meta/
      main.yml
```

### Using Roles
```yaml
- hosts: appservers
  roles:
    - common
    - { role: myapp, app_version: "2.1" }
    - role: monitoring
      vars:
        monitoring_interval: 30
```

### Galaxy (community roles)
```bash
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install -r requirements.yml
ansible-galaxy init myrole                 # scaffold new role
```

---

## Ansible Vault

```bash
# Create encrypted file
ansible-vault create secrets.yml

# Encrypt existing file
ansible-vault encrypt vars/secrets.yml

# Edit encrypted file
ansible-vault edit vars/secrets.yml

# View encrypted file
ansible-vault view vars/secrets.yml

# Decrypt
ansible-vault decrypt vars/secrets.yml

# Encrypt a single value (inline)
ansible-vault encrypt_string 'mypassword' --name 'db_password'

# Run playbook with vault
ansible-playbook site.yml --vault-password-file ~/.vault_pass
ansible-playbook site.yml --ask-vault-pass
```

---

## Jinja2 Templates

```jinja
# templates/app.conf.j2
[server]
host = {{ ansible_default_ipv4.address }}
port = {{ app_port | default(8080) }}
workers = {{ ansible_processor_vcpus * 2 }}
environment = {{ env | upper }}

{% if enable_ssl %}
ssl_cert = /etc/ssl/certs/app.crt
ssl_key = /etc/ssl/private/app.key
{% endif %}

{% for backend in backends %}
upstream {{ backend.name }} {{ backend.host }}:{{ backend.port }};
{% endfor %}
```

---

## Best Practices

1. **Idempotency**: Tasks should be safe to run multiple times with the same result
2. **Tags**: Tag tasks for selective execution (`--tags`, `--skip-tags`)
3. **Check mode**: Always test with `--check` + `--diff` before applying
4. **Vault for secrets**: Never store plaintext passwords in playbooks or vars
5. **Roles over monolithic playbooks**: Promotes reuse
6. **Version pin modules**: Avoid `state: latest` in production
7. **Limit execution**: Use `--limit` for targeted rollouts
8. **Handler deduplication**: Multiple notifies trigger handler only once per play

---

## Common Interview Questions

### Q1: What makes Ansible agentless?
Ansible uses SSH (or WinRM) to connect to managed nodes. Only Python is required on targets. No persistent daemon runs on managed hosts.

### Q2: What is idempotency?
Running a playbook multiple times produces the same end state without unintended side effects. Ansible modules are designed to check current state before making changes.

### Q3: Difference between `command` and `shell` modules?
- `command`: Executes a command directly — no shell features (`|`, `>`, variables)
- `shell`: Runs through `/bin/sh` — supports pipes, redirects, globs

### Q4: What are handlers and when are they triggered?
Handlers are tasks that run only when notified by another task that reports a `changed` state. They execute at the end of the play (not immediately), and only once even if notified multiple times.

### Q5: How does Ansible determine task changed vs ok?
Each module checks the current state of the resource against the desired state. If they match: `ok`. If a change was made: `changed`. Some modules (command, shell) always report `changed` unless you use `changed_when`.

### Q6: What is `become`?
Privilege escalation — equivalent to `sudo`. `become: true` runs tasks as root (or another user via `become_user`).

### Q7: How do you make a task always/never report changed?
```yaml
- shell: date
  changed_when: false        # always report ok

- command: /app/check.sh
  changed_when: "'CHANGED' in result.stdout"
  register: result
```

### Q8: Difference between static and dynamic includes?
- `import_tasks` / `import_playbook`: Parsed at load time (static) — conditions evaluated at parse time
- `include_tasks`: Processed at runtime (dynamic) — supports loops and runtime conditions

### Q9: How would you manage secrets in Ansible?
Use Ansible Vault to encrypt sensitive files or individual values. Vault password can be stored in a file (not in version control) or fetched from a password manager/secret store at runtime.

### Q10: How does Ansible handle failures?
- `ignore_errors: true` — continue on failure
- `failed_when` — custom failure conditions
- `block` / `rescue` / `always` — try/catch/finally equivalent
- `max_fail_percentage` — allow a percentage of hosts to fail before aborting
- `any_errors_fatal: true` — abort all hosts if any fails
