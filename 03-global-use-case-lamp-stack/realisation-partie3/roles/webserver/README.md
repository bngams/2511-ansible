# Webserver Role

## Description
This role configures an Apache web server with PHP support on multiple Linux distributions (Debian/Ubuntu and Alpine Linux).

## Requirements
- Ansible 2.9 or higher
- `community.general` collection (for Alpine Linux support)

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
# Web server document root directory
web_document_root: /var/www/html

# Apache service name
apache_service_name: apache2

# Directory permissions for document root
web_directory_mode: '0755'
```

Variables needed from your playbook or inventory:
- `mysql_dbname`: Database name (used in template)
- `mysql_user`: Database username (used in template)
- `mysql_password`: Database password (used in template)
- `mysql_host`: Database host address (used in template)

## Dependencies
None.

## Example Playbook

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  gather_facts: true  # Required to detect OS family

  vars_files:
    - vars/main.yml

  roles:
    - webserver
```

## Directory Structure
```
webserver/
├── README.md           # This file
├── defaults/
│   └── main.yml       # Default variables
├── files/
│   └── app/           # Web application files
├── handlers/
│   └── main.yml       # Handlers (if needed)
├── meta/
│   └── main.yml       # Role metadata
├── tasks/
│   └── main.yml       # Main tasks
├── templates/
│   └── db-config.php.j2  # Database configuration template
└── vars/
    └── main.yml       # Role variables
```

## License
MIT

## Author Information
Created for Aelion training purposes.
