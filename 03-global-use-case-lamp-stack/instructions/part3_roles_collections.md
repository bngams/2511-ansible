# Organisation avancée avec les Rôles et Collections Ansible

## Introduction

Dans les tutoriels précédents, nous avons créé un playbook Ansible fonctionnel pour déployer une stack LAMP. Cependant, à mesure que vos projets grandissent, maintenir un seul fichier playbook devient complexe et difficile à gérer.

**Problématiques rencontrées avec un playbook monolithique :**

- 📁 **Difficulté de maintenance** : fichier unique de plus en plus long
- 🔄 **Réutilisabilité limitée** : impossible de réutiliser des portions de code dans d'autres projets
- 👥 **Collaboration complexe** : plusieurs personnes travaillant sur le même fichier
- 🧪 **Tests difficiles** : tester des parties isolées devient compliqué
- 📚 **Organisation peu claire** : tout est mélangé dans un seul endroit

**Solution : Les Rôles Ansible**

Les rôles permettent d'organiser votre code Ansible en composants réutilisables, testables et maintenables. Chaque rôle encapsule une fonctionnalité spécifique (installer un serveur web, configurer une base de données, etc.).

**Objectifs de ce tutoriel :**

1. Transformer notre playbook monolithique en rôles modulaires
2. Créer deux rôles : `webserver` et `database`
3. Apprendre à déboguer et tester nos rôles
4. Découvrir Ansible Galaxy et réutiliser des rôles communautaires

## Comprendre les Rôles Ansible

### Qu'est-ce qu'un rôle ?

Un **rôle** est une structure organisée de fichiers et de répertoires qui regroupe des tâches, variables, fichiers, templates et handlers liés à une fonctionnalité précise.

**Avantages des rôles :**

✅ **Modularité** : chaque rôle est indépendant et réutilisable  
✅ **Lisibilité** : structure claire et standardisée  
✅ **Testabilité** : possibilité de tester chaque rôle isolément  
✅ **Partage** : facilement partageable via Ansible Galaxy  
✅ **Maintenance** : modifications localisées et simples  

### Structure standard d'un rôle

```
roles/
└── nom_du_role/
    ├── tasks/           # Tâches principales (obligatoire)
    │   └── main.yml
    ├── handlers/        # Handlers (optionnel)
    │   └── main.yml
    ├── templates/       # Templates Jinja2 (optionnel)
    │   └── config.j2
    ├── files/           # Fichiers statiques (optionnel)
    │   └── app.conf
    ├── vars/            # Variables (optionnel)
    │   └── main.yml
    ├── defaults/        # Variables par défaut (optionnel)
    │   └── main.yml
    ├── meta/            # Métadonnées du rôle (optionnel)
    │   └── main.yml
    └── tests/           # Tests du rôle (optionnel)
        ├── inventory
        └── test.yml
```

**Description des répertoires :**

| Répertoire | Description | Priorité |
|------------|-------------|----------|
| `tasks/` | Contient les tâches à exécuter (fichier `main.yml` obligatoire) | **Obligatoire** |
| `handlers/` | Contient les handlers (actions déclenchées par notify) | Optionnel |
| `templates/` | Contient les templates Jinja2 (fichiers `.j2`) | Optionnel |
| `files/` | Contient les fichiers statiques à copier | Optionnel |
| `vars/` | Variables du rôle (haute priorité) | Optionnel |
| `defaults/` | Variables par défaut (basse priorité, écrasables) | Optionnel |
| `meta/` | Métadonnées : dépendances, auteur, version, etc. | Optionnel |
| `tests/` | Tests du rôle (inventaire et playbook de test) | Optionnel |

:::info Différence vars/ vs defaults/
- **`defaults/`** : variables par défaut, priorité la plus basse, facilement écrasables
- **`vars/`** : variables du rôle, priorité plus élevée, difficiles à écraser

**Règle d'or :** utilisez `defaults/` pour les valeurs configurables par l'utilisateur, et `vars/` pour les valeurs internes au rôle.
:::

### Utilisation d'un rôle dans un playbook

Une fois votre rôle créé, vous pouvez l'utiliser dans un playbook :

```yaml
---
- hosts: web
  become: true
  roles:
    - webserver
    - common

- hosts: db
  become: true
  roles:
    - database
    - common
```

Ansible cherchera automatiquement les rôles dans le répertoire `roles/` à la racine de votre projet.

## Transformation du Playbook en Rôles

### Analyse du playbook actuel

Avant de commencer la transformation, analysons notre playbook pour identifier les rôles à créer :

**Playbook actuel :**
```
playbook.yml
├── Play 1 : web servers
│   ├── Installation Apache/PHP
│   ├── Configuration répertoire web
│   ├── Déploiement application
│   └── Démarrage service Apache
│
└── Play 2 : database servers
    ├── Installation MySQL/MariaDB
    ├── Configuration mot de passe
    ├── Autorisation connexions externes
    ├── Création base de données
    └── Création utilisateur
```

**Rôles à créer :**
- **`webserver`** : gère tout ce qui concerne le serveur web
- **`database`** : gère tout ce qui concerne la base de données

### Création de la structure des rôles

**Étape 1 : Créer la structure de répertoires**

Ansible fournit une commande pour générer automatiquement la structure d'un rôle :

```bash
# Se placer à la racine du projet
cd /chemin/vers/projet

# Créer le rôle webserver
ansible-galaxy role init roles/webserver

# Créer le rôle database
ansible-galaxy role init roles/database
```

**Résultat :**
```
roles/
├── webserver/
│   ├── tasks/
│   │   └── main.yml
│   ├── handlers/
│   │   └── main.yml
│   ├── templates/
│   ├── files/
│   ├── vars/
│   │   └── main.yml
│   ├── defaults/
│   │   └── main.yml
│   ├── meta/
│   │   └── main.yml
│   └── tests/
│       ├── inventory
│       └── test.yml
│
└── database/
    └── (même structure)
```

:::info Alternative manuelle
Si vous préférez créer la structure manuellement :
```bash
mkdir -p roles/webserver/{tasks,handlers,templates,files,vars,defaults,meta,tests}
touch roles/webserver/tasks/main.yml
touch roles/webserver/handlers/main.yml
# ... répéter pour database
```
:::

### Transformation du rôle webserver

#### Étape 1 : Identifier les composants à extraire

Du playbook original, identifiez :
- ✅ Les tâches → `tasks/main.yml`
- ✅ Les handlers → `handlers/main.yml`
- ✅ Les templates → `templates/`
- ✅ Les fichiers → `files/`
- ✅ Les variables → `defaults/main.yml`

#### Étape 2 : Déplacer les tâches

**Ancien playbook (extrait partie web) :**
```yaml
- hosts: web
  become: true
  vars_files: vars/main.yml

  tasks:
    - name: install apache and php last version
      apt:
        name:
          - apache2
          - php
          - php-mysql
        state: present
        update_cache: yes

    - name: Give writable mode to http folder
      file:
        path: /var/www/html
        state: directory
        mode: '0755'
    # ... autres tâches
```

**Nouveau fichier `roles/webserver/tasks/main.yml` :**

```yaml
---
# roles/webserver/tasks/main.yml

- name: install apache and php last version (Debian family)
  apt:
    name:
      - apache2
      - php
      - php-mysql
    state: present
    update_cache: yes
  when: ansible_facts['os_family'] == "Debian"

- name: install apache and php last version (RedHat family)
  yum:
    name:
      - httpd
      - php
      - php-mysqlnd
    state: present
    update_cache: yes
  when: ansible_facts['os_family'] == "RedHat"

- name: Give writable mode to http folder
  file:
    path: /var/www/html
    state: directory
    mode: '0755'

- name: remove default index.html
  file:
    path: /var/www/html/index.html
    state: absent

- name: upload web app source
  copy:
    src: app/
    dest: /var/www/html/

- name: deploy php database config
  template:
    src: db-config.php.j2
    dest: /var/www/html/db-config.php

- name: ensure apache service is started (Debian family)
  service:
    name: apache2
    state: started
    enabled: yes
  when: ansible_facts['os_family'] == "Debian"
  notify: restart apache

- name: ensure apache service is started (RedHat family)
  service:
    name: httpd
    state: started
    enabled: yes
  when: ansible_facts['os_family'] == "RedHat"
  notify: restart apache

- name: enable connection with remote database (RedHat family)
  shell: setsebool -P httpd_can_network_connect_db 1
  when: ansible_facts['os_family'] == "RedHat"
```

**📝 Note importante :** Les chemins `src:` dans les modules `copy` et `template` sont maintenant relatifs au rôle. Ansible cherchera automatiquement dans `roles/webserver/files/` et `roles/webserver/templates/`.

#### Étape 3 : Déplacer les handlers

**Fichier `roles/webserver/handlers/main.yml` :**

```yaml
---
# roles/webserver/handlers/main.yml

- name: restart apache
  service:
    name: "{{ 'apache2' if ansible_facts['os_family'] == 'Debian' else 'httpd' }}"
    state: restarted
```

#### Étape 4 : Déplacer les fichiers et templates

```bash
# Déplacer les fichiers sources de l'application
mv files/app/* roles/webserver/files/

# Déplacer le template de configuration
mv templates/db-config.php.j2 roles/webserver/templates/
```

**Structure après déplacement :**
```
roles/webserver/
├── files/
│   ├── index.php
│   └── validation.php
└── templates/
    └── db-config.php.j2
```

#### Étape 5 : Définir les variables par défaut

**Fichier `roles/webserver/defaults/main.yml` :**

```yaml
---
# roles/webserver/defaults/main.yml
# Variables par défaut pour le rôle webserver

# Configuration base de données (à écraser dans le playbook ou inventaire)
db_host: "192.168.0.22"
mysql_dbname: "blog"
mysql_user: "admin"
mysql_password: "secret"

# Configuration serveur web
web_root: "/var/www/html"
web_port: 80
```

:::warning Variables sensibles
Les variables comme `mysql_password` ne devraient pas être en clair. Utilisez Ansible Vault pour les sécuriser dans un environnement de production.
:::

#### Étape 6 : Ajouter les métadonnées

**Fichier `roles/webserver/meta/main.yml` :**

```yaml
---
# roles/webserver/meta/main.yml

galaxy_info:
  author: Votre Nom
  description: Installation et configuration d'un serveur web Apache/PHP
  company: Votre Entreprise
  
  license: MIT
  
  min_ansible_version: 2.9
  
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: Debian
      versions:
        - buster
        - bullseye
    - name: EL
      versions:
        - 8
        - 9
  
  galaxy_tags:
    - web
    - apache
    - php
    - lamp

dependencies: []
```

### Transformation du rôle database

#### Structure complète du rôle database

Suivez la même méthodologie que pour le rôle webserver :

**Fichier `roles/database/tasks/main.yml` :**

```yaml
---
# roles/database/tasks/main.yml

- name: install mysql (Debian family)
  apt:
    name:
      - mysql-server
      - python3-pymysql
    state: present
    update_cache: yes
  when: ansible_facts['os_family'] == "Debian"

- name: install mariadb (RedHat family)
  yum:
    name:
      - mariadb-server
      - python3-PyMySQL
    state: present
    update_cache: yes
  when: ansible_facts['os_family'] == "RedHat"

- name: ensure database service is started
  service:
    name: "{{ 'mysql' if ansible_facts['os_family'] == 'Debian' else 'mariadb' }}"
    state: started
    enabled: yes

- name: Create MySQL client config
  copy:
    dest: "/root/.my.cnf"
    content: |
      [client]
      user=root
      password={{ root_password }}
    mode: 0400

- name: Allow external MySQL connections (Debian)
  lineinfile:
    path: /etc/mysql/mysql.conf.d/mysqld.cnf
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
  loop:
    - { regexp: '^skip-external-locking', line: '# skip-external-locking' }
    - { regexp: '^bind-address', line: '# bind-address' }
  when: ansible_facts['os_family'] == "Debian"
  notify: restart database

- name: Allow external MySQL connections (RedHat)
  lineinfile:
    path: /etc/my.cnf
    regexp: '^bind-address'
    line: "bind-address = 0.0.0.0"
    insertafter: '^\[mysqld\]'
  when: ansible_facts['os_family'] == "RedHat"
  notify: restart database

- name: upload sql table config
  template:
    src: table.sql.j2
    dest: /tmp/table.sql

- name: create database
  mysql_db:
    name: "{{ mysql_dbname }}"
    state: present
    login_user: root
    login_password: "{{ root_password }}"

- name: import database schema
  mysql_db:
    name: "{{ mysql_dbname }}"
    state: import
    target: /tmp/table.sql
    login_user: root
    login_password: "{{ root_password }}"

- name: create database user
  mysql_user:
    name: "{{ mysql_user }}"
    password: "{{ mysql_password }}"
    priv: "{{ mysql_dbname }}.*:ALL"
    host: "{{ webserver_host }}"
    state: present
    login_user: root
    login_password: "{{ root_password }}"
    login_unix_socket: "{{ mysql_socket }}"
```

**Fichier `roles/database/handlers/main.yml` :**

```yaml
---
# roles/database/handlers/main.yml

- name: restart database
  service:
    name: "{{ 'mysql' if ansible_facts['os_family'] == 'Debian' else 'mariadb' }}"
    state: restarted
```

**Fichier `roles/database/defaults/main.yml` :**

```yaml
---
# roles/database/defaults/main.yml

# Configuration base de données
mysql_dbname: "blog"
mysql_user: "admin"
mysql_password: "secret"
root_password: "my_secret_password"
webserver_host: "192.168.0.21"

# Chemins spécifiques selon l'OS (définis dans vars/)
mysql_socket: "{{ '/var/run/mysqld/mysqld.sock' if ansible_facts['os_family'] == 'Debian' else '/var/lib/mysql/mysql.sock' }}"
```

**Déplacer le template SQL :**

```bash
mv templates/table.sql.j2 roles/database/templates/
```

### Nouveau playbook utilisant les rôles

**Fichier `playbook.yml` (nouvelle version) :**

```yaml
---
# Déploiement serveur web
- hosts: web
  become: true
  roles:
    - webserver

# Déploiement serveur base de données
- hosts: db
  become: true
  roles:
    - database
```

**C'est tout !** Le playbook est maintenant beaucoup plus simple et lisible. Toute la logique est encapsulée dans les rôles.

### Structure finale du projet

```
projet-ansible/
├── ansible.cfg
├── inventory.yml
├── playbook.yml
├── vars/
│   └── main.yml                  # Variables globales (si nécessaire)
└── roles/
    ├── webserver/
    │   ├── tasks/
    │   │   └── main.yml
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── templates/
    │   │   └── db-config.php.j2
    │   ├── files/
    │   │   ├── index.php
    │   │   └── validation.php
    │   ├── defaults/
    │   │   └── main.yml
    │   └── meta/
    │       └── main.yml
    │
    └── database/
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── templates/
        │   └── table.sql.j2
        ├── defaults/
        │   └── main.yml
        └── meta/
            └── main.yml
```

### Exécution du nouveau playbook

```bash
# Exécution normale
ansible-playbook playbook.yml

# Exécuter seulement le rôle webserver
ansible-playbook playbook.yml --tags webserver

# Exécuter seulement le rôle database
ansible-playbook playbook.yml --tags database
```

:::info Utilisation des tags
Pour filtrer l'exécution par rôle, ajoutez des tags dans votre playbook :
```yaml
- hosts: web
  become: true
  roles:
    - role: webserver
      tags: webserver
```
:::

## Débogage des Playbooks et Rôles

### Introduction au débogage

Le débogage est une étape essentielle dans le développement de playbooks Ansible. Plusieurs techniques et outils permettent d'identifier et de résoudre les problèmes.

**Ressources complètes sur le débogage :**

Pour un guide complet et détaillé sur le débogage des playbooks Ansible, consultez l'article dédié :  
👉 [**Déboguer vos playbooks Ansible**](https://devopssec.fr/article/deboguer-playbooks-ansible#begin-article-section)

Cet article couvre :
- Le mode verbeux (`-v`, `-vv`, `-vvv`, `-vvvv`)
- Le module `debug` et ses utilisations avancées
- Le mode check (`--check`) et diff (`--diff`)
- Le débogage des templates Jinja2
- L'analyse des erreurs courantes
- Les stratégies de débogage efficaces

### Techniques de débogage rapide

#### 1. Mode verbeux

Plus vous ajoutez de `v`, plus la sortie est détaillée :

```bash
# Verbosité niveau 1 (affiche les résultats des tâches)
ansible-playbook playbook.yml -v

# Verbosité niveau 2 (affiche les paramètres des modules)
ansible-playbook playbook.yml -vv

# Verbosité niveau 3 (affiche les connexions SSH)
ansible-playbook playbook.yml -vvv

# Verbosité niveau 4 (affiche les détails internes)
ansible-playbook playbook.yml -vvvv
```

#### 2. Module debug

Insérez des points de débogage dans vos rôles :

```yaml
# roles/webserver/tasks/main.yml
- name: Debug - Afficher les variables
  debug:
    msg: |
      OS Family: {{ ansible_facts['os_family'] }}
      Database Host: {{ db_host }}
      MySQL Database: {{ mysql_dbname }}

- name: Debug - Afficher le contenu d'une variable
  debug:
    var: ansible_facts

- name: Debug - Condition
  debug:
    msg: "Cette tâche s'exécute seulement sur Debian"
  when: ansible_facts['os_family'] == "Debian"
```

#### 3. Module assert

Validez vos conditions et variables :

```yaml
# roles/database/tasks/main.yml
- name: Vérifier que les variables requises sont définies
  assert:
    that:
      - mysql_dbname is defined
      - mysql_user is defined
      - mysql_password is defined
      - root_password is defined
    fail_msg: "Variables requises manquantes pour le rôle database"
    success_msg: "Toutes les variables requises sont définies"
```

#### 4. Mode check et diff

Simule l'exécution sans appliquer les changements :

```bash
# Mode check (dry-run)
ansible-playbook playbook.yml --check

# Mode diff (affiche les changements avant/après)
ansible-playbook playbook.yml --diff

# Combinaison check + diff
ansible-playbook playbook.yml --check --diff
```

#### 5. Stratégie de débogage : module fail

Arrêter le playbook pour inspecter l'état :

```yaml
- name: Installation des packages
  apt:
    name: apache2
    state: present

- name: STOP ICI POUR DEBUG
  fail:
    msg: "Arrêt volontaire pour inspection"
  when: true  # Changez en false pour désactiver
```

### Débogage spécifique aux rôles

#### Tester un rôle isolément

Créez un playbook de test minimal :

**Fichier `test-webserver.yml` :**

```yaml
---
- hosts: localhost
  become: true
  vars:
    db_host: "localhost"
    mysql_dbname: "test_blog"
    mysql_user: "test_user"
    mysql_password: "test_password"
  
  roles:
    - webserver
```

Exécution :
```bash
ansible-playbook test-webserver.yml --check
```

#### Lister les tâches d'un rôle

```bash
# Lister toutes les tâches du playbook
ansible-playbook playbook.yml --list-tasks

# Lister les tags disponibles
ansible-playbook playbook.yml --list-tags
```

#### Démarrer à une tâche spécifique

```bash
# Démarrer à partir d'une tâche précise
ansible-playbook playbook.yml --start-at-task="deploy php database config"
```

## Tests Automatisés avec Molecule

### Introduction à Molecule

**Molecule** est un framework de test pour Ansible qui permet de :
- ✅ Tester vos rôles dans différents environnements
- ✅ Automatiser les tests unitaires et d'intégration
- ✅ Valider l'idempotence de vos rôles
- ✅ Tester sur plusieurs distributions simultanément

**Installation de Molecule :**

```bash
# Installation avec pip
pip install molecule molecule-docker

# Vérifier l'installation
molecule --version
```

:::info Prérequis
- Python 3.6+
- Docker (pour les tests en conteneurs)
- pip et virtualenv
:::

### Initialisation de Molecule pour un rôle

**Étape 1 : Initialiser Molecule dans le rôle webserver**

```bash
# Se placer dans le répertoire du rôle
cd roles/webserver

# Initialiser Molecule avec le driver Docker
molecule init scenario --driver-name docker
```

**Structure créée :**

```
roles/webserver/
└── molecule/
    └── default/
        ├── converge.yml       # Playbook de test
        ├── molecule.yml       # Configuration Molecule
        └── verify.yml         # Tests de vérification
```

### Configuration de Molecule

**Fichier `molecule/default/molecule.yml` :**

```yaml
---
# roles/webserver/molecule/default/molecule.yml

dependency:
  name: galaxy

driver:
  name: docker

platforms:
  - name: webserver-ubuntu
    image: geerlingguy/docker-ubuntu2004-ansible:latest
    pre_build_image: true
    privileged: true
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    tmpfs:
      - /run
      - /tmp

  - name: webserver-centos
    image: geerlingguy/docker-centos8-ansible:latest
    pre_build_image: true
    privileged: true
    command: /usr/sbin/init
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    tmpfs:
      - /run
      - /tmp

provisioner:
  name: ansible
  inventory:
    host_vars:
      webserver-ubuntu:
        db_host: "192.168.0.22"
        mysql_dbname: "test_blog"
        mysql_user: "test_user"
        mysql_password: "test_password"
      webserver-centos:
        db_host: "192.168.0.22"
        mysql_dbname: "test_blog"
        mysql_user: "test_user"
        mysql_password: "test_password"

verifier:
  name: ansible
```

:::info Images Docker pour Ansible
Les images `geerlingguy/docker-*-ansible` sont spécialement conçues pour tester des rôles Ansible. Elles incluent systemd et les outils nécessaires.
:::

### Playbook de test

**Fichier `molecule/default/converge.yml` :**

```yaml
---
# roles/webserver/molecule/default/converge.yml

- name: Converge
  hosts: all
  become: true
  
  vars:
    db_host: "192.168.0.22"
    mysql_dbname: "test_blog"
    mysql_user: "test_user"
    mysql_password: "test_password"
  
  tasks:
    - name: "Include webserver role"
      include_role:
        name: webserver
```

### Tests de vérification

**Fichier `molecule/default/verify.yml` :**

```yaml
---
# roles/webserver/molecule/default/verify.yml

- name: Verify
  hosts: all
  become: true
  
  tasks:
    - name: Check if Apache is installed (Debian)
      package:
        name: apache2
        state: present
      check_mode: yes
      register: apache_debian
      when: ansible_facts['os_family'] == "Debian"
      failed_when: apache_debian is changed

    - name: Check if Apache is installed (RedHat)
      package:
        name: httpd
        state: present
      check_mode: yes
      register: apache_redhat
      when: ansible_facts['os_family'] == "RedHat"
      failed_when: apache_redhat is changed

    - name: Check if Apache service is running (Debian)
      service:
        name: apache2
        state: started
      check_mode: yes
      register: apache_service_debian
      when: ansible_facts['os_family'] == "Debian"
      failed_when: apache_service_debian is changed

    - name: Check if Apache service is running (RedHat)
      service:
        name: httpd
        state: started
      check_mode: yes
      register: apache_service_redhat
      when: ansible_facts['os_family'] == "RedHat"
      failed_when: apache_service_redhat is changed

    - name: Verify web root directory exists
      stat:
        path: /var/www/html
      register: web_root
      failed_when: not web_root.stat.exists

    - name: Verify web root has correct permissions
      stat:
        path: /var/www/html
      register: web_root_perms
      failed_when: web_root_perms.stat.mode != '0755'

    - name: Check if PHP is installed
      command: php --version
      register: php_version
      changed_when: false
      failed_when: php_version.rc != 0

    - name: Verify application files are present
      stat:
        path: "/var/www/html/{{ item }}"
      register: app_files
      loop:
        - index.php
        - validation.php
        - db-config.php
      failed_when: not app_files.results | map(attribute='stat.exists') | list | min

    - name: Test HTTP response
      uri:
        url: http://localhost
        status_code: 200
      register: http_response
      failed_when: http_response.status != 200
```

### Exécution des tests Molecule

**Cycle de vie complet :**

```bash
# Se placer dans le répertoire du rôle
cd roles/webserver

# 1. Créer l'instance de test
molecule create

# 2. Préparer l'instance (installations de base)
molecule prepare

# 3. Exécuter le rôle (converge)
molecule converge

# 4. Vérifier l'idempotence (le rôle ne doit rien changer à la 2ème exécution)
molecule idempotence

# 5. Exécuter les tests de vérification
molecule verify

# 6. Détruire l'instance
molecule destroy

# OU exécuter tout le cycle en une commande
molecule test
```

**Commandes utiles :**

```bash
# Lister les instances Molecule
molecule list

# Se connecter à une instance pour déboguer
molecule login --host webserver-ubuntu

# Exécuter seulement certaines étapes
molecule converge  # Applique le rôle
molecule verify    # Lance les tests
```

### Exemple de test ciblé : tâche d'installation Apache

Créons un test spécifique pour vérifier l'installation d'Apache :

**Fichier `molecule/default/verify-apache.yml` :**

```yaml
---
# roles/webserver/molecule/default/verify-apache.yml

- name: Verify Apache Installation
  hosts: all
  become: true
  
  tasks:
    - name: Gather package facts
      package_facts:
        manager: auto

    - name: Assert Apache is installed (Debian)
      assert:
        that:
          - "'apache2' in ansible_facts.packages"
        fail_msg: "Apache2 is not installed on Debian system"
        success_msg: "Apache2 is correctly installed"
      when: ansible_facts['os_family'] == "Debian"

    - name: Assert Apache is installed (RedHat)
      assert:
        that:
          - "'httpd' in ansible_facts.packages"
        fail_msg: "Apache (httpd) is not installed on RedHat system"
        success_msg: "Apache (httpd) is correctly installed"
      when: ansible_facts['os_family'] == "RedHat"

    - name: Get Apache service status
      service_facts:

    - name: Assert Apache service is running and enabled (Debian)
      assert:
        that:
          - "ansible_facts.services['apache2.service'].state == 'running'"
          - "ansible_facts.services['apache2.service'].status == 'enabled'"
        fail_msg: "Apache2 service is not running or not enabled"
        success_msg: "Apache2 service is running and enabled"
      when: ansible_facts['os_family'] == "Debian"

    - name: Assert Apache service is running and enabled (RedHat)
      assert:
        that:
          - "ansible_facts.services['httpd.service'].state == 'running'"
          - "ansible_facts.services['httpd.service'].status == 'enabled'"
        fail_msg: "Apache (httpd) service is not running or not enabled"
        success_msg: "Apache (httpd) service is running and enabled"
      when: ansible_facts['os_family'] == "RedHat"

    - name: Test Apache is listening on port 80
      wait_for:
        port: 80
        timeout: 5
      register: port_check
      failed_when: port_check is failed
```

**Exécution du test ciblé :**

```bash
# Modifier molecule.yml pour utiliser ce fichier de vérification
# Puis exécuter
molecule verify
```

### Tests avec Testinfra (alternative Python)

Molecule peut aussi utiliser **Testinfra** pour écrire des tests en Python :

**Installation :**
```bash
pip install pytest-testinfra
```

**Fichier `molecule/default/tests/test_webserver.py` :**

```python
"""
Test suite for webserver role
"""

import os
import pytest
import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']
).get_hosts('all')


def test_apache_is_installed(host):
    """Test that Apache is installed"""
    if host.system_info.distribution == 'ubuntu':
        apache = host.package("apache2")
    else:
        apache = host.package("httpd")
    
    assert apache.is_installed


def test_apache_is_running(host):
    """Test that Apache service is running"""
    if host.system_info.distribution == 'ubuntu':
        apache = host.service("apache2")
    else:
        apache = host.service("httpd")
    
    assert apache.is_running
    assert apache.is_enabled


def test_php_is_installed(host):
    """Test that PHP is installed"""
    cmd = host.run("php --version")
    assert cmd.rc == 0


def test_web_root_exists(host):
    """Test that web root directory exists"""
    web_root = host.file("/var/www/html")
    assert web_root.exists
    assert web_root.is_directory
    assert web_root.mode == 0o755


def test_application_files_exist(host):
    """Test that application files are deployed"""
    files = [
        "/var/www/html/index.php",
        "/var/www/html/validation.php",
        "/var/www/html/db-config.php"
    ]
    
    for file in files:
        f = host.file(file)
        assert f.exists
        assert f.is_file


def test_apache_listening_on_port_80(host):
    """Test that Apache is listening on port 80"""
    assert host.socket("tcp://0.0.0.0:80").is_listening


def test_http_response(host):
    """Test HTTP response from Apache"""
    cmd = host.run("curl -s -o /dev/null -w '%{http_code}' http://localhost")
    assert cmd.stdout.strip() == "200"
```

**Configuration Molecule pour Testinfra :**

Modifier `molecule/default/molecule.yml` :

```yaml
verifier:
  name: testinfra
  options:
    v: 1
```

**Exécution :**
```bash
molecule verify
```

**Sortie attendue :**
```
============================= test session starts ==============================
collected 8 items

tests/test_webserver.py::test_apache_is_installed PASSED             [ 12%]
tests/test_webserver.py::test_apache_is_running PASSED               [ 25%]
tests/test_webserver.py::test_php_is_installed PASSED                [ 37%]
tests/test_webserver.py::test_web_root_exists PASSED                 [ 50%]
tests/test_webserver.py::test_application_files_exist PASSED         [ 62%]
tests/test_webserver.py::test_apache_listening_on_port_80 PASSED     [ 75%]
tests/test_webserver.py::test_http_response PASSED                   [ 87%]

======================== 8 passed in 5.23s =================================
```

### Intégration Continue avec Molecule

**Exemple avec GitHub Actions :**

**Fichier `.github/workflows/molecule.yml` :**

```yaml
name: Molecule Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        role: [webserver, database]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          pip install molecule molecule-docker docker

      - name: Run Molecule tests
        run: |
          cd roles/${{ matrix.role }}
          molecule test
```

## Utiliser des Rôles depuis Ansible Galaxy

### Qu'est-ce qu'Ansible Galaxy ?

**Ansible Galaxy** est le dépôt communautaire officiel de rôles Ansible. C'est l'équivalent de Docker Hub pour Docker, ou PyPI pour Python.

**Site web :** [https://galaxy.ansible.com/](https://galaxy.ansible.com/)

**Avantages :**
- ✅ Accès à des milliers de rôles prêts à l'emploi
- ✅ Rôles maintenus par la communauté
- ✅ Notation et commentaires des utilisateurs
- ✅ Installation simple avec la CLI Ansible Galaxy
- ✅ Gain de temps considérable

### Rechercher un rôle sur Galaxy

**Méthode 1 : Via le site web**

1. Visitez [galaxy.ansible.com](https://galaxy.ansible.com/)
2. Utilisez la barre de recherche
3. Filtrez par :
   - Note (étoiles)
   - Nombre de téléchargements
   - Date de dernière mise à jour
   - Plateformes supportées

**Méthode 2 : Via la ligne de commande**

```bash
# Rechercher un rôle
ansible-galaxy search nginx

# Rechercher avec filtres
ansible-galaxy search nginx --author geerlingguy --platforms EL
```

### Exemple pratique : Créer un nouveau projet avec des rôles Galaxy

Imaginons que nous voulons créer un nouveau projet pour déployer :
- Un serveur Nginx (au lieu d'Apache)
- Une base PostgreSQL (au lieu de MySQL)
- Un certificat SSL avec Let's Encrypt

**Étape 1 : Rechercher les rôles appropriés**

```bash
# Rechercher un rôle Nginx
ansible-galaxy search nginx --author geerlingguy

# Rechercher un rôle PostgreSQL
ansible-galaxy search postgresql --author geerlingguy

# Rechercher un rôle certbot (Let's Encrypt)
ansible-galaxy search certbot --author geerlingguy
```

**Rôles recommandés (très populaires et maintenus) :**
- `geerlingguy.nginx` : Installation et configuration de Nginx
- `geerlingguy.postgresql` : Installation de PostgreSQL
- `geerlingguy.certbot` : Gestion des certificats SSL Let's Encrypt

**Étape 2 : Consulter la documentation des rôles**

Avant d'installer, consultez toujours :
- Le README sur GitHub
- Les variables disponibles (`defaults/main.yml`)
- Les exemples d'utilisation
- La compatibilité des distributions

**Exemple pour geerlingguy.nginx :**
- GitHub : https://github.com/geerlingguy/ansible-role-nginx
- Galaxy : https://galaxy.ansible.com/geerlingguy/nginx

**Étape 3 : Créer la structure du nouveau projet**

```bash
# Créer le répertoire du projet
mkdir projet-nginx-postgresql
cd projet-nginx-postgresql

# Créer la structure
mkdir -p group_vars host_vars
touch ansible.cfg inventory.yml playbook.yml requirements.yml
```

**Étape 4 : Définir les rôles requis**

**Fichier `requirements.yml` :**

```yaml
---
# requirements.yml
# Liste des rôles à installer depuis Ansible Galaxy

roles:
  # Rôle Nginx
  - name: geerlingguy.nginx
    version: 3.1.4

  # Rôle PostgreSQL
  - name: geerlingguy.postgresql
    version: 3.4.0

  # Rôle Certbot (Let's Encrypt)
  - name: geerlingguy.certbot
    version: 6.1.0

collections:
  # Collection pour les modules PostgreSQL
  - name: community.postgresql
    version: 2.4.0
```

:::info Spécifier une version
Il est recommandé de spécifier une version précise pour garantir la reproductibilité. Sans version, Ansible installera la dernière version disponible.
:::

**Étape 5 : Installer les rôles**

```bash
# Installer tous les rôles définis dans requirements.yml
ansible-galaxy install -r requirements.yml

# Installer dans un répertoire personnalisé
ansible-galaxy install -r requirements.yml -p ./roles

# Forcer la réinstallation (utile pour les mises à jour)
ansible-galaxy install -r requirements.yml --force
```

**Vérification :**
```bash
# Lister les rôles installés
ansible-galaxy list

# Afficher les informations d'un rôle
ansible-galaxy info geerlingguy.nginx
```

**Étape 6 : Créer l'inventaire**

**Fichier `inventory.yml` :**

```yaml
all:
  children:
    webservers:
      hosts:
        web-01:
          ansible_host: 192.168.0.30
        web-02:
          ansible_host: 192.168.0.31
    
    databases:
      hosts:
        db-01:
          ansible_host: 192.168.0.40
  
  vars:
    ansible_user: root
```

**Étape 7 : Configurer les variables**

**Fichier `group_vars/webservers.yml` :**

```yaml
---
# Variables pour les serveurs web

# Configuration Nginx
nginx_remove_default_vhost: true
nginx_vhosts:
  - listen: "80"
    server_name: "example.com www.example.com"
    root: "/var/www/html"
    index: "index.php index.html"
    state: "present"
    template: "{{ nginx_vhost_template }}"
    extra_parameters: |
      location ~ \.php$ {
          fastcgi_pass unix:/var/run/php/php-fpm.sock;
          fastcgi_index index.php;
          fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
          include fastcgi_params;
      }

# Configuration Certbot
certbot_auto_renew: true
certbot_auto_renew_hour: "3"
certbot_auto_renew_minute: "30"
certbot_create_if_missing: true
certbot_admin_email: admin@example.com
certbot_certs:
  - domains:
      - example.com
      - www.example.com

# Configuration PHP (si nécessaire)
php_version: "8.1"
php_packages:
  - php{{ php_version }}-fpm
  - php{{ php_version }}-cli
  - php{{ php_version }}-pgsql
```

**Fichier `group_vars/databases.yml` :**

```yaml
---
# Variables pour les serveurs de base de données

# Configuration PostgreSQL
postgresql_version: "14"
postgresql_databases:
  - name: myapp_db
    encoding: UTF8
    lc_collate: en_US.UTF-8
    lc_ctype: en_US.UTF-8

postgresql_users:
  - name: myapp_user
    password: "{{ vault_db_password }}"
    encrypted: yes
    priv: "myapp_db:ALL"
    role_attr_flags: NOCREATEDB,NOSUPERUSER

postgresql_hba_entries:
  - type: host
    database: myapp_db
    user: myapp_user
    address: 192.168.0.0/24
    auth_method: md5

# Autoriser les connexions externes
postgresql_listen_addresses: "*"
```

**Étape 8 : Créer le playbook**

**Fichier `playbook.yml` :**

```yaml
---
# Déploiement des serveurs web avec Nginx
- name: Configure web servers
  hosts: webservers
  become: true
  
  pre_tasks:
    - name: Update apt cache (Debian)
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == 'Debian'
  
  roles:
    - role: geerlingguy.nginx
    - role: geerlingguy.certbot
  
  post_tasks:
    - name: Ensure web root exists
      file:
        path: /var/www/html
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

# Déploiement des serveurs de base de données
- name: Configure database servers
  hosts: databases
  become: true
  
  pre_tasks:
    - name: Update apt cache (Debian)
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == 'Debian'
  
  roles:
    - role: geerlingguy.postgresql
  
  post_tasks:
    - name: Ensure PostgreSQL is started
      service:
        name: postgresql
        state: started
        enabled: yes
```

**Étape 9 : Sécuriser les mots de passe avec Vault**

```bash
# Créer un fichier vault pour les variables sensibles
ansible-vault create group_vars/databases/vault.yml
```

**Contenu de `group_vars/databases/vault.yml` :**
```yaml
---
vault_db_password: "super_secret_password_here"
```

**Étape 10 : Exécuter le playbook**

```bash
# Vérification de la syntaxe
ansible-playbook playbook.yml --syntax-check

# Mode dry-run
ansible-playbook playbook.yml --check

# Exécution réelle
ansible-playbook playbook.yml --ask-vault-pass

# Ou avec un fichier de mot de passe vault
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass.txt
```

### Personnaliser un rôle Galaxy

Parfois, un rôle Galaxy ne correspond pas exactement à vos besoins. Voici comment le personnaliser :

**Option 1 : Surcharger les variables**

La plupart des rôles Galaxy sont hautement configurables via les variables.

```yaml
# group_vars/webservers.yml
nginx_user: "nginx"
nginx_worker_processes: "auto"
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_client_max_body_size: "64m"
```

**Option 2 : Utiliser les hooks (pre_tasks / post_tasks)**

```yaml
- hosts: webservers
  become: true
  
  pre_tasks:
    - name: Installer des packages supplémentaires
      apt:
        name:
          - vim
          - htop
        state: present
  
  roles:
    - geerlingguy.nginx
  
  post_tasks:
    - name: Ajouter une configuration personnalisée
      copy:
        content: |
          # Configuration custom
          server_tokens off;
        dest: /etc/nginx/conf.d/custom.conf
      notify: restart nginx
```

**Option 3 : Wrapper le rôle dans votre propre rôle**

Créez un rôle qui encapsule le rôle Galaxy :

```yaml
# roles/my-nginx/meta/main.yml
---
dependencies:
  - role: geerlingguy.nginx
    vars:
      nginx_remove_default_vhost: true
```

```yaml
# roles/my-nginx/tasks/main.yml
---
- name: Configurations personnalisées après Nginx
  template:
    src: custom-site.conf.j2
    dest: /etc/nginx/sites-available/custom-site.conf
  notify: restart nginx
```

### Contribuer à Ansible Galaxy

Vous pouvez publier vos propres rôles sur Galaxy :

**Étape 1 : Créer un compte sur galaxy.ansible.com**

**Étape 2 : Préparer votre rôle**

```bash
# Vérifier que le rôle respecte les standards
ansible-galaxy role init my-awesome-role

# Remplir le fichier meta/main.yml avec les informations
```

**Étape 3 : Publier sur GitHub**

Ansible Galaxy s'intègre avec GitHub. Poussez votre rôle sur un dépôt GitHub.

**Étape 4 : Importer sur Galaxy**

1. Connectez-vous sur galaxy.ansible.com
2. Allez dans "My Content"
3. Cliquez sur "Add Content"
4. Sélectionnez votre dépôt GitHub

**Étape 5 : Maintenir votre rôle**

À chaque push sur GitHub, Galaxy peut automatiquement importer la nouvelle version.

## Bonnes Pratiques des Rôles

### 1. Nommage cohérent

```
✅ roles/webserver      # Bon
✅ roles/database       # Bon
✅ roles/common         # Bon

❌ roles/web_srv        # Éviter les underscores
❌ roles/WebServer      # Éviter le camelCase
❌ roles/db-setup       # Éviter les tirets (utiliser underscores si nécessaire)
```

### 2. Variables avec préfixes

Pour éviter les collisions de variables entre rôles :

```yaml
# ❌ Mauvais
database_name: blog
user: admin

# ✅ Bon
mysql_database_name: blog
mysql_user: admin
```

### 3. Documentation dans README.md

Chaque rôle doit avoir un README.md complet :

```markdown
# Role Name: webserver

## Description
Installation et configuration d'un serveur web Apache/Nginx avec PHP.

## Requirements
- Ansible >= 2.9
- Distributions supportées : Ubuntu 20.04+, Debian 10+, CentOS 8+

## Role Variables

### Required Variables
- `db_host` : Adresse IP du serveur de base de données
- `mysql_dbname` : Nom de la base de données

### Optional Variables
- `web_root` : Répertoire racine du site web (défaut: `/var/www/html`)
- `web_port` : Port d'écoute (défaut: `80`)

## Dependencies
Aucune dépendance externe.

## Example Playbook

\`\`\`yaml
- hosts: webservers
  become: true
  vars:
    db_host: "192.168.0.22"
    mysql_dbname: "myapp"
  roles:
    - webserver
\`\`\`

## License
MIT

## Author
Votre Nom <email@example.com>
```

### 4. Utiliser des tags

Ajoutez des tags pour exécuter des parties spécifiques :

```yaml
# roles/webserver/tasks/main.yml
- name: Install packages
  apt:
    name: apache2
    state: present
  tags:
    - webserver
    - packages
    - install

- name: Configure Apache
  template:
    src: apache.conf.j2
    dest: /etc/apache2/apache2.conf
  tags:
    - webserver
    - config
```

**Utilisation :**
```bash
# Exécuter seulement les tâches d'installation
ansible-playbook playbook.yml --tags install

# Exécuter seulement les tâches de configuration
ansible-playbook playbook.yml --tags config

# Exclure certains tags
ansible-playbook playbook.yml --skip-tags config
```

### 5. Idempotence

Assurez-vous que vos rôles sont idempotents :

```yaml
# ❌ Mauvais (pas idempotent)
- name: Add line to config
  shell: echo "ServerName localhost" >> /etc/apache2/apache2.conf

# ✅ Bon (idempotent)
- name: Set ServerName in Apache config
  lineinfile:
    path: /etc/apache2/apache2.conf
    regexp: '^ServerName'
    line: 'ServerName localhost'
```

### 6. Gestion des secrets

Ne jamais stocker de secrets en clair :

```yaml
# ❌ Mauvais
defaults/main.yml:
  mysql_password: "password123"

# ✅ Bon
defaults/main.yml:
  mysql_password: "{{ vault_mysql_password }}"

group_vars/all/vault.yml (chiffré):
  vault_mysql_password: !vault |
    $ANSIBLE_VAULT;1.1;AES256
    ...
```

### 7. Dépendances de rôles

Déclarez les dépendances dans `meta/main.yml` :

```yaml
# roles/webserver/meta/main.yml
---
dependencies:
  - role: common
    vars:
      common_packages:
        - vim
        - curl
  - role: firewall
    vars:
      firewall_allowed_tcp_ports:
        - 80
        - 443
```

## Conclusion

Dans ce tutoriel complet, nous avons abordé :

### 1. **Organisation avec les Rôles**
- ✅ Comprendre la structure d'un rôle Ansible
- ✅ Transformer un playbook monolithique en rôles modulaires
- ✅ Créer les rôles `webserver` et `database`
- ✅ Organiser le code de manière professionnelle

### 2. **Débogage et Tests**
- ✅ Techniques de débogage avec le mode verbeux et le module `debug`
- ✅ Utilisation de `assert` et du mode `--check`
- ✅ Ressources complètes sur le débogage Ansible
- ✅ Tests automatisés avec Molecule
- ✅ Tests unitaires avec Testinfra (Python)
- ✅ Validation de l'idempotence

### 3. **Ansible Galaxy**
- ✅ Rechercher et installer des rôles communautaires
- ✅ Créer un projet avec des rôles Galaxy
- ✅ Personnaliser et étendre les rôles existants
- ✅ Publier ses propres rôles

### 4. **Bonnes Pratiques**
- ✅ Nommage cohérent et conventions
- ✅ Documentation complète (README)
- ✅ Utilisation des tags
- ✅ Gestion sécurisée des secrets avec Vault
- ✅ Garantir l'idempotence

### Avantages de l'approche par rôles

📦 **Modularité** : code organisé en composants réutilisables  
🔄 **Réutilisabilité** : partage facile entre projets  
🧪 **Testabilité** : tests isolés de chaque rôle  
👥 **Collaboration** : travail en équipe facilité  
📚 **Maintenabilité** : modifications localisées  
🚀 **Productivité** : utilisation de rôles communautaires  

### Prochaines étapes

Pour aller encore plus loin :

1. **Collections Ansible** : découvrir les collections pour organiser modules, plugins et rôles
2. **Ansible Tower / AWX** : interface web pour gérer vos playbooks en entreprise
3. **Dynamic Inventory** : inventaires dynamiques depuis cloud providers (AWS, Azure, GCP)
4. **Ansible Vault avancé** : rotation des secrets, intégration avec HashiCorp Vault
5. **CI/CD avec Ansible** : intégration dans pipelines GitLab CI, GitHub Actions, Jenkins

### Ressources recommandées

- 📖 [Documentation officielle Ansible](https://docs.ansible.com/)
- 🎓 [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- 🌟 [Ansible Galaxy](https://galaxy.ansible.com/)
- 🧪 [Molecule Documentation](https://molecule.readthedocs.io/)
- 🔧 [Testinfra Documentation](https://testinfra.readthedocs.io/)
- 🐛 [Déboguer vos playbooks Ansible](https://devopssec.fr/article/deboguer-playbooks-ansible#begin-article-section)
- 📦 [Ansible Collections](https://docs.ansible.com/ansible/latest/user_guide/collections_using.html)

Vous êtes maintenant équipé pour créer, tester et maintenir des rôles Ansible professionnels ! 🎉

:::tip Conseil final
Commencez petit : transformez un playbook simple en rôle, testez-le avec Molecule, puis progressivement adoptez ces pratiques dans tous vos projets. La courbe d'apprentissage en vaut la peine !
:::
