# LAMP Stack Deployment with Ansible Roles

## Overview
This project demonstrates how to deploy a complete LAMP (Linux, Apache, MySQL, PHP) stack using **Ansible Roles** for better organization and reusability.

## What's New: Using Roles Instead of Tasks

### Before (realisation-partie2): All Tasks in Playbook
```
playbook.yml  (200+ lines with all tasks mixed together)
├── Web server tasks
├── Database tasks
├── Templates
└── Files
```

### After (realisation-partie3): Organized with Roles
```
playbook.yml           (Clean, ~50 lines, high-level orchestration)
roles/
├── webserver/         (Everything related to web server)
│   ├── tasks/
│   ├── templates/
│   ├── files/
│   ├── defaults/
│   └── meta/
└── database/          (Everything related to database)
    ├── tasks/
    ├── templates/
    ├── handlers/
    ├── defaults/
    └── meta/
```

## Benefits of Using Roles

1. **Reusability**: Use the same role across multiple projects
2. **Organization**: Keep related content together (tasks, templates, files)
3. **Maintainability**: Easier to find and update specific components
4. **Sharing**: Share roles via Ansible Galaxy or Git repositories
5. **Testing**: Test roles independently with tools like Molecule
6. **Separation of Concerns**: Each role has one clear responsibility

## Project Structure

```
realisation-partie3/
│
├── playbook.yml                # Main playbook (orchestrates roles)
├── ansible.cfg                 # Ansible configuration
├── requirements.yml            # Required Ansible collections
├── setup.yml                   # Initial setup tasks
│
├── inventory/                  # Inventory files defining hosts
│   └── hosts.yml
│
├── vars/                       # Variables
│   └── main.yml                # Shared variables (DB config, etc.)
│
├── roles/                      # ROLES DIRECTORY
│   │
│   ├── webserver/              # WEB SERVER ROLE
│   │   ├── tasks/
│   │   │   └── main.yml        # Install Apache+PHP, deploy app
│   │   ├── templates/
│   │   │   └── db-config.php.j2  # Database config template
│   │   ├── files/
│   │   │   └── app/            # Web application files
│   │   ├── defaults/
│   │   │   └── main.yml        # Default variables
│   │   ├── meta/
│   │   │   └── main.yml        # Role metadata
│   │   └── README.md           # Role documentation
│   │
│   └── database/               # DATABASE ROLE
│       ├── tasks/
│       │   └── main.yml        # Install MySQL, create DB/user
│       ├── templates/
│       │   └── table.sql.j2    # Database schema template
│       ├── handlers/
│       │   └── main.yml        # MySQL restart handler
│       ├── defaults/
│       │   └── main.yml        # Default variables
│       ├── meta/
│       │   └── main.yml        # Role metadata
│       └── README.md           # Role documentation
│
└── vault-pass.txt.sample       # Sample vault password file
```

## Understanding Ansible Roles

### What is a Role?

A **role** is a standardized way to organize Ansible content. It bundles:
- **Tasks**: Actions to perform (install packages, copy files, etc.)
- **Handlers**: Actions triggered by task notifications (restart services)
- **Templates**: Jinja2 files with variables
- **Files**: Static files to copy
- **Vars/Defaults**: Variables with different precedence levels
- **Meta**: Role dependencies and information

### Role Directory Structure

Each role follows this standard structure:

```
role_name/
├── tasks/              # Main list of tasks to execute
│   └── main.yml
├── handlers/           # Handlers (triggered by 'notify')
│   └── main.yml
├── templates/          # Jinja2 templates (*.j2)
│   └── config.j2
├── files/              # Static files to copy
│   └── app.conf
├── vars/               # Variables (high precedence)
│   └── main.yml
├── defaults/           # Default variables (lowest precedence)
│   └── main.yml
├── meta/               # Role metadata and dependencies
│   └── main.yml
└── README.md           # Documentation
```

Ansible automatically loads these files when a role is applied!

## The Two Roles

### 1. Webserver Role

**Purpose**: Configure Apache web server with PHP support

**What it does**:
- Installs Apache and PHP (supports Debian/Ubuntu and Alpine)
- Configures document root permissions
- Deploys web application files
- Generates PHP database configuration from template
- Starts and enables Apache service

**Key files**:
- `tasks/main.yml`: Installation and configuration tasks
- `templates/db-config.php.j2`: PHP database connection config
- `files/app/`: Web application source code

### 2. Database Role

**Purpose**: Configure MariaDB/MySQL database server

**What it does**:
- Installs MariaDB server and Python MySQL libraries
- Sets secure root password (from Ansible Vault)
- Configures MySQL for remote connections
- Creates application database and imports schema
- Creates dedicated database user with privileges
- Restarts MySQL when configuration changes (via handler)

**Key files**:
- `tasks/main.yml`: Database installation and configuration
- `handlers/main.yml`: MySQL restart handler
- `templates/table.sql.j2`: Database schema template

## How to Use This Project

### Prerequisites

1. Install Ansible:
   ```bash
   pip install ansible
   ```

2. Install required collections:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

3. Configure inventory in `inventory/hosts.yml`:
   ```yaml
   all:
     children:
       web:
         hosts:
           webserver1:
             ansible_host: 192.168.1.10
       db:
         hosts:
           dbserver1:
             ansible_host: 192.168.1.20
   ```

4. Set variables in `vars/main.yml`:
   ```yaml
   mysql_dbname: myapp_db
   mysql_user: myapp_user
   mysql_password: "{{ vault_mysql_password }}"
   webserver_host: "192.168.1.10"
   ```

5. Create vault for passwords:
   ```bash
   ansible-vault create vars/secrets.yml
   ```

### Run the Playbook

```bash
# Basic execution
ansible-playbook playbook.yml --ask-vault-pass

# With specific inventory
ansible-playbook -i inventory/hosts.yml playbook.yml --ask-vault-pass

# Check mode (dry-run, no changes)
ansible-playbook playbook.yml --ask-vault-pass --check

# Limit to specific hosts
ansible-playbook playbook.yml --ask-vault-pass --limit web

# Verbose output (for debugging)
ansible-playbook playbook.yml --ask-vault-pass -vvv
```

## Key Concepts for Students

### 1. Role Structure
Every role follows the same directory layout, making it easy to navigate and understand.

### 2. Variable Precedence
From lowest to highest priority:
1. `role/defaults/main.yml` - Lowest
2. `vars/main.yml` (vars_files)
3. `playbook vars:` section
4. Command line `-e` (extra vars) - Highest

### 3. Handlers
Special tasks that:
- Only run when notified by other tasks
- Only run if the notifying task reports "changed"
- Run at the end of the play
- Example: Restart MySQL only when config changes

### 4. Templates (Jinja2)
Files with `.j2` extension containing variables:
```jinja2
<?php
$db_host = "{{ mysql_host }}";
$db_name = "{{ mysql_dbname }}";
?>
```

### 5. Ansible Vault
Encrypt sensitive data (passwords, API keys):
```bash
# Encrypt a string
ansible-vault encrypt_string 'mypassword' --name 'mysql_password'

# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml
```

### 6. Multi-Distribution Support
Use `gather_facts: true` + `when` conditions:
```yaml
- name: Install Apache (Debian)
  apt:
    name: apache2
  when: ansible_facts['os_family'] == "Debian"

- name: Install Apache (Alpine)
  apk:
    name: apache2
  when: ansible_facts['os_family'] == "Alpine"
```

### 7. Idempotency
Ansible is idempotent - safe to run multiple times:
- First run: Makes changes
- Subsequent runs: Reports "ok" (no changes needed)

## Comparison with Previous Approach

| Aspect | Without Roles (partie2) | With Roles (partie3) |
|--------|------------------------|---------------------|
| **Organization** | All tasks in one file | Separated by function |
| **Reusability** | Copy-paste tasks | Import role |
| **Maintenance** | Hard to find tasks | Clear structure |
| **Testing** | Test entire playbook | Test individual roles |
| **Sharing** | Share entire project | Share specific roles |
| **Scalability** | Gets messy at scale | Stays organized |

## Advanced Topics

### Role Dependencies
Roles can depend on other roles (in `meta/main.yml`):
```yaml
dependencies:
  - role: common
  - role: firewall
    vars:
      open_ports: [80, 443]
```

### Ansible Galaxy
Share or download roles from the community:
```bash
# Download role from Galaxy
ansible-galaxy install geerlingguy.apache

# Create role skeleton
ansible-galaxy init my_new_role
```

### Role Variables
Pass variables to roles:
```yaml
roles:
  - role: webserver
    vars:
      apache_port: 8080
      php_version: "8.2"
```

## Troubleshooting

### Common Issues

1. **Module not found (apk, mysql_user)**
   - Solution: Install required collections: `ansible-galaxy collection install -r requirements.yml`

2. **Vault password error**
   - Solution: Provide vault password with `--ask-vault-pass` or `--vault-password-file`

3. **Template not found**
   - Solution: Ensure templates are in `role/templates/` directory

4. **Permission denied**
   - Solution: Use `become: true` in playbook

5. **MySQL connection refused**
   - Solution: Check firewall rules, ensure MySQL allows remote connections

## Learning Resources

- [Ansible Roles Documentation](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)

## License
MIT - For educational purposes (Aelion Training)

## Author
Aelion Training Team
