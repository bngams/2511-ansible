# Database Role

## Description
This role installs and configures MariaDB/MySQL database server with support for remote connections. It handles database creation, schema import, and user management with proper security practices.

## Requirements
- Ansible 2.9 or higher
- Collections:
  - `community.mysql` (for database management modules)
  - `community.general` (for Alpine Linux support)

Install required collections:
```bash
ansible-galaxy collection install community.mysql community.general
```

## Role Variables

### Required Variables
These variables **must** be provided by your playbook or inventory:

```yaml
# MySQL root password (use Ansible Vault!)
root_password: "your_secure_root_password"

# Application database name
mysql_dbname: "myapp_db"

# Application database user
mysql_user: "myapp_user"

# Application database password (use Ansible Vault!)
mysql_password: "secure_app_password"

# Web server hostname or IP (for access control)
webserver_host: "192.168.1.10"
```

### Optional Variables (with defaults)
See `defaults/main.yml` for customizable settings:

```yaml
# MySQL service name (auto-detected based on OS)
mysql_service_name: mysql

# Configuration file paths (distribution-specific)
mysql_config_path_debian: /etc/mysql/mariadb.conf.d/50-server.cnf
mysql_config_path_alpine: /etc/my.cnf.d/mariadb-server.cnf
```

## Dependencies
None.

## Example Playbook

### Basic Usage
```yaml
---
- name: Configure database server
  hosts: db
  become: true
  gather_facts: true

  vars_files:
    - vars/secrets.yml  # Contains vault-encrypted passwords

  roles:
    - role: database
      vars:
        mysql_dbname: webapp_db
        mysql_user: webapp_user
        mysql_password: "{{ vault_mysql_password }}"
        root_password: "{{ vault_root_password }}"
        webserver_host: "192.168.1.100"
```

### Using Ansible Vault for Passwords
Create an encrypted file for sensitive data:
```bash
ansible-vault create vars/secrets.yml
```

Contents of `vars/secrets.yml`:
```yaml
---
vault_root_password: "super_secure_root_password"
vault_mysql_password: "secure_app_password"
```

Run playbook with vault password:
```bash
ansible-playbook playbook.yml --ask-vault-pass
```

## What This Role Does

1. **Install MariaDB Server**
   - Installs database server and required Python libraries
   - Supports Debian/Ubuntu (apt) and Alpine (apk)

2. **Configure MySQL Root Access**
   - Sets secure root password
   - Creates `/root/.my.cnf` for automatic authentication

3. **Enable Remote Connections**
   - Modifies configuration to allow external access
   - Comments out `bind-address` and `skip-external-locking`

4. **Create Database and Schema**
   - Creates application database
   - Imports SQL schema from template

5. **Create Application User**
   - Creates dedicated database user
   - Grants privileges only on application database
   - Restricts access to specific web server host

## Security Notes

- **Always use Ansible Vault** for passwords
- The role restricts database user access to specified `webserver_host`
- Root password is stored in `/root/.my.cnf` with `0400` permissions
- In production, consider additional firewall rules

## Directory Structure
```
database/
├── README.md           # This file
├── defaults/
│   └── main.yml       # Default variables
├── handlers/
│   └── main.yml       # MySQL restart handler
├── meta/
│   └── main.yml       # Role metadata and dependencies
├── tasks/
│   └── main.yml       # Main tasks
├── templates/
│   └── table.sql.j2   # Database schema template
└── vars/
    └── main.yml       # Role variables (if needed)
```

## License
MIT

## Author Information
Created for Aelion training purposes.
