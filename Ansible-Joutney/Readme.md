# Ansible Learning Guide

A comprehensive guide to understanding and implementing Ansible for configuration management and automation.

## Table of Contents
1. [Introduction to Configuration Management](#introduction-to-configuration-management)
2. [Prerequisites](#prerequisites)
3. [SSH Key Setup](#ssh-key-setup)
4. [Ansible Installation](#ansible-installation)
5. [Inventory Management](#inventory-management)
6. [Ad-hoc Commands](#ad-hoc-commands)
7. [Ansible Playbooks](#ansible-playbooks)
8. [Common Modules](#common-modules)
9. [Variables and Facts](#variables-and-facts)
10. [Handlers](#handlers)
11. [Roles](#roles)
12. [Best Practices](#best-practices)
13. [Troubleshooting](#troubleshooting)

---

## Introduction to Configuration Management

**Configuration Management (CM)** is a systematic approach to maintaining the consistency of a system's approach to maintaining consistency of a system's performance, functionality, and physical attributes.

### Purpose of CM/Ansible

- **Automate** the process of provisioning, updating, and managing software and system configuration
- **Ensure** that all the machines (servers, VMs, containers) are setup and configured exactly how we want
- In DevOps, it refers to maintaining the state of servers and infrastructure
- It's about ensuring that all the machines are setup and configured exactly how we want

### Before Configuration Management Tools

System administrators had to:
- Manually SSH into every machine
- Run different commands for different OS (Windows, Linux, CentOS)
- Time-consuming and error-prone process

### Benefits of Ansible

- **Agentless**: No need to install agents on target machines
- **Idempotent**: Running the same playbook multiple times produces the same result
- **Simple**: Uses YAML syntax which is easy to read and write
- **Powerful**: Can manage thousands of servers efficiently

---

## Prerequisites

- Basic understanding of Linux commands
- SSH access to target machines
- Python installed on control node and managed nodes
- Root or sudo access on target machines

---
## To Install Ansible on the Host or any other devise 
```bash
sudo apt update
 #Updates the local package index so the system is aware of the latest available packages.

sudo apt install ansible
#Installs Ansible from the system’s package repository. This method is simple and recommended for beginners.
```

## SSH Key Setup

SSH keys provide a secure way to authenticate without passwords.

### Understanding SSH Keys

- **ssh-keygen**: Used to generate public and private key pairs
- **Private Key**: Kept secret on your local machine
- **Public Key**: Shared with target machines

### Generating SSH Keys

```bash
# Generate a new SSH key pair
ssh-keygen

# The command will create:
# - Private key: ~/.ssh/id_rsa (keep this secret!)
# - Public key: ~/.ssh/id_rsa.pub (share this)
```

### Setting Up Passwordless Authentication

The private key is used to login without a password.

```bash
# Copy public key to target machine
ssh-copy-content id_rsa.pub ubuntu@PublicIPAddress

# Or manually:
# 1. Copy content of id_rsa.pub
# 2. On target machine, paste into ~/.ssh/authorized_keys
```

### Testing Connection

```bash
# SSH to target machine (should not ask for password)
ssh ubuntu@<target-ip>

# Alternatively, using private key explicitly
ssh -i ~/.ssh/id_rsa ubuntu@<target-ip>
```

**Note**: `.ssh` is a hidden file. To see it, use `ls -la`

---

## Ansible Installation

### On Ubuntu/Debian

```bash
# Update package list
sudo apt update

# Install Ansible
sudo apt install ansible

# Verify installation
ansible --version
```

---

## Inventory Management

The **inventory file** contains information about target machines (hosts) where Ansible will execute tasks.

### Creating an Inventory File

Create a file called `inventory.ini`:

```ini
# Simple inventory with IP addresses
[webservers]
192.168.x.xx
192.168.x.xx

[databases]
192.168.x.x

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```
 
### Inventory File Structure

```ini
# Group name in square brackets
[group_name]
host1 ansible_host=192.168.1.10
host2 ansible_host=192.168.1.11

# Define variables for a group
[group_name:vars]
ansible_user=ubuntu
ansible_port=22
```

### Using the Inventory File

```bash
# Test connectivity to all hosts
ansible -i inventory.ini all -m ping

# Target specific group
ansible -i inventory.ini webservers -m ping
```

---

## Ad-hoc Commands

Ad-hoc commands are one-liners for quick tasks without writing a playbook.

### Basic Syntax

```bash
ansible -i <inventory> <host-pattern> -m <module> -a "<arguments>"
```

### Common Ad-hoc Commands

```bash
# Ping all hosts
ansible -i inventory.ini all -m ping

# Check disk space
ansible -i inventory.ini all -m shell -a "df -h"

# Install a package
ansible -i inventory.ini webservers -m apt -a "name=nginx state=present" --become

# Restart a service
ansible -i inventory.ini webservers -m service -a "name=nginx state=restarted" --become

# Copy a file
ansible -i inventory.ini all -m copy -a "src=/local/file dest=/remote/file"

# Create a directory
ansible -i inventory.ini all -m file -a "path=/tmp/mydir state=directory"
```

### Common Options

- `-i`: Specify inventory file
- `-m`: Module to use
- `-a`: Arguments for the module
- `--become`: Execute with sudo privileges
- `-b`: Short form of --become

---

## Ansible Playbooks

Playbooks are YAML files that define a series of tasks to be executed on target hosts.

### Basic Playbook Structure

```yaml
---
# playbook.yml
- name: Install and start nginx
  hosts: webservers
  become: yes
  
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    
    - name: Start nginx service
      service:
        name: nginx
        state: started
```

### Running a Playbook

```bash
# Execute playbook
ansible-playbook -i inventory.ini playbook.yml

# With specific tags
ansible-playbook -i inventory.ini playbook.yml --tags "install"

# Dry run (check mode)
ansible-playbook -i inventory.ini playbook.yml --check
```

### Playbook Components

#### Hosts
Defines which machines to target:
```yaml
hosts: all           # All hosts
hosts: webservers    # Specific group
hosts: server1       # Single host
```

#### Variables
```yaml
vars:
  http_port: 80
  max_clients: 200

tasks:
  - name: Configure nginx
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
```

#### Tasks
Each task uses a module:
```yaml
tasks:
  - name: Ensure nginx is installed
    apt:
      name: nginx
      state: present
      
  - name: Copy configuration file
    copy:
      src: files/nginx.conf
      dest: /etc/nginx/nginx.conf
      owner: root
      group: root
      mode: '0644'
```

### States in Ansible

- **present**: Ensure something exists (install if not present)
- **absent**: Ensure something doesn't exist (remove if present)
- **started**: Ensure service is running
- **stopped**: Ensure service is stopped
- **restarted**: Restart the service
- **enabled**: Enable service to start on boot

### Example: Complete Playbook

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  
  vars:
    nginx_port: 80
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
    
    - name: Install nginx
      apt:
        name: nginx
        state: present
    
    - name: Start and enable nginx
      service:
        name: nginx
        state: started
        enabled: yes
    
    - name: Copy index.html
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
```

---

## Common Modules

### Shell Module
Execute shell commands:
```yaml
- name: Run shell command
  shell: df -h
  register: disk_space

- name: Display output
  debug:
    var: disk_space.stdout
```

### Apt Module (Debian/Ubuntu)
Manage packages:
```yaml
- name: Install multiple packages
  apt:
    name:
      - nginx
      - git
      - curl
    state: present
    update_cache: yes
```

### Copy Module
Copy files from local to remote:
```yaml
- name: Copy configuration file
  copy:
    src: files/config.conf
    dest: /etc/myapp/config.conf
    backup: yes
```

### Template Module
Copy files with variable substitution:
```yaml
- name: Deploy configuration from template
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
```

### File Module
Manage files and directories:
```yaml
- name: Create a directory
  file:
    path: /opt/myapp
    state: directory
    mode: '0755'
    owner: ubuntu
    group: ubuntu
```

### Service Module
Manage services:
```yaml
- name: Ensure nginx is running
  service:
    name: nginx
    state: started
    enabled: yes
```

---

## Variables and Facts

### Defining Variables

**In playbook:**
```yaml
vars:
  app_name: myapp
  app_port: 8080

tasks:
  - name: Create app directory
    file:
      path: "/opt/{{ app_name }}"
      state: directory
```

**In separate file (vars.yml):**
```yaml
# vars.yml
app_name: myapp
app_port: 8080
database_name: myappdb
```

**Using variable file:**
```yaml
- name: Deploy application
  hosts: webservers
  vars_files:
    - vars.yml
  
  tasks:
    - name: Print app name
      debug:
        msg: "Application name is {{ app_name }}"
```

### Gathering Facts

Ansible automatically collects system information (facts):
```yaml
- name: Display facts
  hosts: all
  tasks:
    - name: Show OS family
      debug:
        msg: "OS Family: {{ ansible_os_family }}"
    
    - name: Show IP address
      debug:
        msg: "IP: {{ ansible_default_ipv4.address }}"
```

---

## Handlers

Handlers are tasks that run only when notified by another task.

```yaml
- name: Configure web server
  hosts: webservers
  become: yes
  
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    
    - name: Copy nginx config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx
  
  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

**Key Points:**
- Handlers run at the end of the playbook
- Only run if notified
- Run only once even if notified multiple times

---

## Roles

Roles help organize playbooks into reusable components.

### Role Directory Structure

```
roles/
└── webserver/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── files/
    │   └── index.html
    ├── vars/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    └── meta/
        └── main.yml
```

### Creating a Role

```bash
# Create role structure
ansible-galaxy init webserver
```

### Using Roles in Playbook

```yaml
- name: Configure servers
  hosts: webservers
  become: yes
  
  roles:
    - webserver
    - firewall
    - monitoring
```

---

## Best Practices

### 1. Directory Structure

```
ansible-project/
├── inventory/
│   ├── production
│   └── staging
├── group_vars/
│   └── all.yml
├── host_vars/
│   └── server1.yml
├── roles/
├── playbooks/
├── files/
├── templates/
└── ansible.cfg
```

### 2. Use Version Control
- Keep your playbooks in Git
- Use meaningful commit messages
- Create branches for different environments

### 3. Naming Conventions
- Use descriptive names for tasks
- Use lowercase with underscores for variables
- Use meaningful playbook names

### 4. Idempotency
- Always write idempotent tasks
- Use appropriate modules instead of shell commands
- Test playbooks multiple times

### 5. Security
- Use Ansible Vault for sensitive data
- Never commit passwords or keys to version control
- Use SSH keys instead of passwords

### 6. Documentation
- Comment your playbooks
- Maintain README files
- Document variables and their purposes

---

## Troubleshooting

### Common Issues and Solutions

**Issue: SSH Connection Failed**
```bash
# Test SSH connection manually
ssh -i ~/.ssh/id_rsa ubuntu@<target-ip>

# Check SSH key permissions
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

**Issue: Permission Denied**
```yaml
# Use become for privilege escalation
- name: Install package
  apt:
    name: nginx
    state: present
  become: yes
```

**Issue: Module Not Found**
```bash
# Check Ansible version
ansible --version

# Update Ansible
sudo apt update && sudo apt upgrade ansible
```

### Verbose Output

```bash
# Run with verbose output
ansible-playbook playbook.yml -v    # Basic
ansible-playbook playbook.yml -vv   # More verbose
ansible-playbook playbook.yml -vvv  # Very verbose
ansible-playbook playbook.yml -vvvv # Debug level
```

### Syntax Check

```bash
# Check playbook syntax
ansible-playbook playbook.yml --syntax-check

# Dry run (check mode)
ansible-playbook playbook.yml --check
```

---

## Quick Reference

### Essential Commands

```bash
# Ping all hosts
ansible all -m ping

# Run ad-hoc command
ansible all -m shell -a "uptime"

# Execute playbook
ansible-playbook playbook.yml

# List hosts
ansible all --list-hosts

# Check syntax
ansible-playbook playbook.yml --syntax-check
```

### Common Modules

- `ping`: Test connectivity
- `shell`: Execute shell commands
- `apt/yum`: Package management
- `copy`: Copy files
- `template`: Copy with variables
- `file`: Manage files/directories
- `service`: Manage services

---

## Contributing

Feel free to contribute to this guide by:
1. Reporting issues
2. Suggesting improvements
3. Adding examples
4. Fixing errors

---

**Happy Automating! 🚀**