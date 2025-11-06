# Important notes

We encrypted the sensitive variables (host IP and password) using Ansible vault in node3.yml:6-20. 

When running playbooks, use:

ansible-playbook playbook.yml -i inventory/hosts.yml --ask-vault-pass

Or with a vault password file:
ansible-playbook playbook.yml -i inventory/hosts.yml --vault-password-file vault-pass.txt

The IDE errors about "Unresolved tag: !vault" are expected - Ansible will handle them correctly at runtime.