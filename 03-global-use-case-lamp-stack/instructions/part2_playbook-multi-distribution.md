# Rendre notre Playbook Ansible multi-distribution

## Introduction

Dans le tutoriel précédent, nous avons créé un playbook Ansible pour déployer une stack LAMP. Cependant, notre playbook actuel est conçu uniquement pour fonctionner sur des systèmes de la famille **Debian/Ubuntu** (utilisation du module `apt`, service `apache2`, etc.).

**Objectif de ce tutoriel :**

Adapter notre playbook pour qu'il soit compatible avec plusieurs distributions Linux, notamment :
- **Famille Debian** : Debian, Ubuntu, Linux Mint, etc.
- **Famille RedHat** : RHEL, CentOS, Rocky Linux, AlmaLinux, Fedora, etc.

:::info Important - Simulation pédagogique
Ce tutoriel est une **simulation pédagogique**. Nous n'allons pas redéployer de nouvelles machines ou de nouveaux nœuds. L'objectif est de vous montrer comment structurer votre playbook pour qu'il s'adapte automatiquement à différentes distributions Linux. Vous apprendrez à utiliser les **Facts** et les **conditions** Ansible pour rendre vos playbooks flexibles et réutilisables.

Si vous souhaitez tester réellement ce playbook, vous devrez disposer de machines avec des distributions différentes dans votre inventaire.
:::

**Pourquoi rendre un playbook multi-distribution ?**

- **Flexibilité** : Un seul playbook pour gérer différents environnements
- **Réutilisabilité** : Pas besoin de maintenir plusieurs versions du même playbook
- **Évolutivité** : Facilite l'ajout de nouvelles distributions
- **Bonnes pratiques** : S'adapte aux infrastructures hétérogènes

## Comprendre les distributions avec les Facts Ansible

### Qu'est-ce qu'un Fact ?

Les **Facts** sont des variables système collectées automatiquement par Ansible sur les hôtes distants lors de l'exécution d'un playbook. Ces informations incluent :
- Le système d'exploitation et sa version
- L'architecture du processeur
- Les adresses IP
- Les montages disques
- Les interfaces réseau
- Et bien plus encore

Ansible collecte ces Facts au début de chaque playbook grâce à la tâche implicite **"Gathering Facts"** que vous avez sûrement déjà aperçue dans les sorties de vos exécutions.

### Récupérer les Facts avec le module setup

Pour afficher tous les Facts d'un hôte, nous pouvons utiliser le module `setup` :

```bash
ansible node-web -m setup
```

**Résultat (extrait) :**

```json
node-web | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.2.15", 
            "192.168.0.21"
        ],
        "ansible_architecture": "x86_64",
        "ansible_distribution": "Ubuntu",
        "ansible_distribution_version": "20.04",
        "ansible_os_family": "Debian",
        "ansible_hostname": "node-web",
        "ansible_virtualization_type": "virtualbox",
        ...
    },
    "changed": false
}
```

La liste complète est très longue. Heureusement, nous pouvons filtrer les résultats.

### Filtrer les Facts

Pour obtenir uniquement les informations sur le système d'exploitation, utilisons le paramètre `filter` :

```bash
ansible node-web -m setup -a "filter=*os*"
```

**Résultat :**

```json
node-web | SUCCESS => {
    "ansible_facts": {
        "ansible_os_family": "Debian",
        "ansible_distribution": "Ubuntu",
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}
```

**La variable clé : `ansible_os_family`**

Cette variable nous indique la famille du système d'exploitation :
- `"Debian"` : pour Debian, Ubuntu, Linux Mint, etc.
- `"RedHat"` : pour RHEL, CentOS, Rocky Linux, AlmaLinux, Fedora, etc.
- `"Suse"` : pour openSUSE, SLES, etc.
- `"Arch"` : pour Arch Linux, Manjaro, etc.

C'est cette variable que nous utiliserons pour adapter notre playbook !

### Tester les Facts dans un playbook

Avant de modifier notre playbook complet, testons l'affichage de la famille d'OS :

```yaml
---
- hosts: web
  become: true
  
  tasks:
    - name: Afficher la famille d'OS
      debug:
        var: ansible_os_family
    
    - name: Arrêter le playbook (pour test)
      fail:
        msg: "Test terminé - Arrêt volontaire"
      when: 1 == 1
```

Exécutons ce test :

```bash
ansible-playbook test-facts.yml
```

**Résultat attendu :**

```
PLAY [web] *******************************************************************

TASK [Gathering Facts] *******************************************************
ok: [node-web]

TASK [Afficher la famille d'OS] **********************************************
ok: [node-web] => {
    "ansible_os_family": "Debian"
}

TASK [Arrêter le playbook (pour test)] ***************************************
fatal: [node-web]: FAILED! => {
    "changed": false,
    "msg": "Test terminé - Arrêt volontaire"
}
```

:::info Astuce de débogage
L'utilisation du module `fail` avec une condition toujours vraie (`when: 1 == 1`) est une astuce pratique pour tester rapidement une ou plusieurs tâches sans exécuter l'intégralité du playbook. Une fois vos tests validés, supprimez simplement cette tâche.
:::

### Syntaxes alternatives pour accéder aux Facts

Il existe deux syntaxes pour accéder aux Facts :

**Syntaxe 1 - Directe :**
```yaml
debug:
  var: ansible_os_family
```

**Syntaxe 2 - Via le dictionnaire ansible_facts (recommandée) :**
```yaml
debug:
  var: ansible_facts['os_family']
```

Les deux fonctionnent, mais la syntaxe avec `ansible_facts` est plus cohérente et évite les conflits de noms de variables.

## Les conditions dans Ansible

### L'instruction `when`

L'instruction `when` permet d'exécuter une tâche uniquement si une condition est remplie. C'est l'élément clé pour rendre notre playbook multi-distribution.

**Syntaxe de base :**

```yaml
- name: Ma tâche
  module_name:
    param: value
  when: condition
```

### Opérateurs de comparaison

Voici les opérateurs disponibles dans les conditions Ansible :

| Opérateur | Description | Exemple | Résultat |
|-----------|-------------|---------|----------|
| `==` | Égalité | `x == 4` | Vrai si x est égal à 4 |
| `!=` | Différence | `x != 4` | Vrai si x est différent de 4 |
| `<` | Strictement inférieur | `x < 4` | Vrai si x est strictement inférieur à 4 |
| `<=` | Inférieur ou égal | `x <= 4` | Vrai si x est inférieur ou égal à 4 |
| `>` | Strictement supérieur | `x > 4` | Vrai si x est strictement supérieur à 4 |
| `>=` | Supérieur ou égal | `x >= 4` | Vrai si x est supérieur ou égal à 4 |
| `is` | Identité d'objet | `x is True` | Vrai si x est True |
| `is not` | Non-identité | `x is not True` | Vrai si x n'est pas True |
| `in` | Appartenance | `x in ['a', 'b']` | Vrai si x est dans la liste |
| `not in` | Non-appartenance | `x not in ['a', 'b']` | Vrai si x n'est pas dans la liste |

### Opérateurs logiques

Vous pouvez combiner plusieurs conditions avec des opérateurs logiques :

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `and` | ET logique | `x == 4 and y == 5` |
| `or` | OU logique | `x == 4 or y == 5` |
| `not` | NON logique | `not (x == 4)` |

### Exemples de conditions

**Exemple 1 - Condition simple :**
```yaml
- name: Installer un package uniquement sur Debian
  apt:
    name: apache2
    state: present
  when: ansible_facts['os_family'] == "Debian"
```

**Exemple 2 - Condition multiple (ET) :**
```yaml
- name: Tâche pour Ubuntu 20.04 uniquement
  debug:
    msg: "Ubuntu 20.04 détecté"
  when: 
    - ansible_facts['distribution'] == "Ubuntu"
    - ansible_facts['distribution_version'] == "20.04"
```

**Exemple 3 - Condition multiple (OU) :**
```yaml
- name: Tâche pour RedHat ou CentOS
  debug:
    msg: "Distribution RedHat détectée"
  when: ansible_facts['os_family'] == "RedHat" or ansible_facts['os_family'] == "CentOS"
```

**Exemple 4 - Vérifier l'existence d'une variable :**
```yaml
- name: Utiliser une variable si elle existe
  debug:
    msg: "La variable existe: {{ ma_variable }}"
  when: ma_variable is defined
```

## Adaptation du Playbook - Partie Serveur Web

### Différences entre Debian et RedHat pour le serveur web

Avant d'adapter le playbook, identifions les différences principales :

| Élément | Debian/Ubuntu | RedHat/CentOS |
|---------|---------------|---------------|
| Gestionnaire de paquets | `apt` | `yum` ou `dnf` |
| Paquet Apache | `apache2` | `httpd` |
| Paquet PHP | `php` | `php` |
| Paquet PHP-MySQL | `php-mysql` | `php-mysqlnd` |
| Service Apache | `apache2` | `httpd` |
| Répertoire web | `/var/www/html` | `/var/www/html` |
| SELinux | Généralement désactivé | Actif par défaut |

### Installation des packages avec conditions

Modifions la tâche d'installation pour supporter les deux familles :

```yaml
tasks:
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
```

### Tâches communes

Certaines tâches restent identiques quelle que soit la distribution :

```yaml
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
      src: "db-config.php.j2"
      dest: "/var/www/html/db-config.php"
```

### Gestion du service Apache

Le nom du service diffère selon la distribution :

```yaml
  - name: ensure apache service is started (Debian family)
    service:
      name: apache2
      state: started
      enabled: yes
    when: ansible_facts['os_family'] == "Debian"

  - name: ensure apache service is started (RedHat family)
    service:
      name: httpd
      state: started
      enabled: yes
    when: ansible_facts['os_family'] == "RedHat"
```

### Configuration SELinux pour RedHat

Sur les systèmes RedHat, SELinux peut bloquer les connexions réseau d'Apache vers la base de données. Il faut autoriser explicitement cette communication :

```yaml
  - name: enable connection with remote database (RedHat family)
    shell: setsebool -P httpd_can_network_connect_db 1
    when: ansible_facts['os_family'] == "RedHat"
```

:::info Qu'est-ce que SELinux ?
SELinux (Security-Enhanced Linux) est un module de sécurité du noyau Linux qui fournit un mécanisme de contrôle d'accès obligatoire (MAC). Sur les distributions RedHat, il est activé par défaut et peut bloquer certaines opérations réseau. La commande `setsebool` permet de modifier ces politiques de sécurité.
:::

### Playbook complet - Partie Web

Voici la partie web complète et adaptée :

```yaml
---
# WEB SERVER
- hosts: web
  become: true
  vars_files: vars/main.yml

  tasks:
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
        src: "db-config.php.j2"
        dest: "/var/www/html/db-config.php"

    - name: ensure apache service is started (Debian family)
      service:
        name: apache2
        state: started
        enabled: yes
      when: ansible_facts['os_family'] == "Debian"

    - name: ensure apache service is started (RedHat family)
      service:
        name: httpd
        state: started
        enabled: yes
      when: ansible_facts['os_family'] == "RedHat"

    - name: enable connection with remote database (RedHat family)
      shell: setsebool -P httpd_can_network_connect_db 1
      when: ansible_facts['os_family'] == "RedHat"
```

## Adaptation du Playbook - Partie Base de données

### Différences entre Debian et RedHat pour MySQL

| Élément | Debian/Ubuntu | RedHat/CentOS |
|---------|---------------|---------------|
| Gestionnaire de paquets | `apt` | `yum` ou `dnf` |
| Paquet serveur | `mysql-server` | `mariadb-server` |
| Paquet client Python | `python-mysqldb` ou `python3-pymysql` | `python3-PyMySQL` |
| Service | `mysql` | `mariadb` |
| Socket Unix | `/var/run/mysqld/mysqld.sock` | `/var/lib/mysql/mysql.sock` |
| Fichier de config | `/etc/mysql/mysql.conf.d/mysqld.cnf` | `/etc/my.cnf` |

:::info MySQL vs MariaDB
Sur les distributions RedHat récentes, MariaDB a remplacé MySQL comme système de gestion de base de données par défaut. MariaDB est un fork de MySQL, maintenu par la communauté, et reste compatible avec MySQL au niveau des commandes et des protocoles.
:::

### Installation des packages base de données

```yaml
tasks:
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
```

### Configuration du mot de passe root

La configuration reste similaire, mais il faut adapter le chemin du socket :

```yaml
  - name: Create MySQL client config
    copy:
      dest: "/root/.my.cnf"
      content: |
        [client]
        user=root
        password={{ root_password }}
      mode: 0400
```

### Autorisation des connexions externes

Les fichiers de configuration diffèrent selon la distribution :

**Pour Debian/Ubuntu :**
```yaml
  - name: Allow external MySQL connections (1/2) - Debian
    lineinfile:
      path: /etc/mysql/mysql.conf.d/mysqld.cnf
      regexp: '^skip-external-locking'
      line: "# skip-external-locking"
    when: ansible_facts['os_family'] == "Debian"
    notify: Restart mysql

  - name: Allow external MySQL connections (2/2) - Debian
    lineinfile:
      path: /etc/mysql/mysql.conf.d/mysqld.cnf
      regexp: '^bind-address'
      line: "# bind-address"
    when: ansible_facts['os_family'] == "Debian"
    notify: Restart mysql
```

**Pour RedHat/CentOS :**
```yaml
  - name: Allow external MySQL connections - RedHat
    lineinfile:
      path: /etc/my.cnf
      regexp: '^bind-address'
      line: "bind-address = 0.0.0.0"
      insertafter: '^\[mysqld\]'
    when: ansible_facts['os_family'] == "RedHat"
    notify: Restart mariadb
```

### Gestion du service base de données

```yaml
  - name: ensure mysql service is started (Debian family)
    service:
      name: mysql
      state: started
      enabled: yes
    when: ansible_facts['os_family'] == "Debian"

  - name: ensure mariadb service is started (RedHat family)
    service:
      name: mariadb
      state: started
      enabled: yes
    when: ansible_facts['os_family'] == "RedHat"
```

### Création de la base de données et de l'utilisateur

Ces tâches nécessitent également une adaptation du socket Unix :

```yaml
  - name: upload sql table config
    template:
      src: "table.sql.j2"
      dest: "/tmp/table.sql"

  - name: add sql table to database
    mysql_db:
      name: "{{ mysql_dbname }}"
      state: present
      login_user: root
      login_password: "{{ root_password }}"
      state: import
      target: /tmp/table.sql

  - name: "Create {{ mysql_user }} with all {{ mysql_dbname }} privileges"
    mysql_user:
      name: "{{ mysql_user }}"
      password: "{{ mysql_password }}"
      priv: "{{ mysql_dbname }}.*:ALL"
      host: "{{ webserver_host }}"
      state: present
      login_user: root
      login_password: "{{ root_password }}"
      login_unix_socket: "{{ '/var/run/mysqld/mysqld.sock' if ansible_facts['os_family'] == 'Debian' else '/var/lib/mysql/mysql.sock' }}"
```

### Handlers adaptés

Il faut créer des handlers différents pour chaque distribution :

```yaml
  handlers:
    - name: Restart mysql
      service:
        name: mysql
        state: restarted
      when: ansible_facts['os_family'] == "Debian"

    - name: Restart mariadb
      service:
        name: mariadb
        state: restarted
      when: ansible_facts['os_family'] == "RedHat"
```

:::warning Attention aux handlers
Les handlers avec des conditions `when` peuvent ne pas se déclencher comme prévu. Une approche alternative consiste à créer un handler unique qui détecte automatiquement le service :

```yaml
handlers:
  - name: Restart database service
    service:
      name: "{{ 'mysql' if ansible_facts['os_family'] == 'Debian' else 'mariadb' }}"
      state: restarted
```
:::

### Playbook complet - Partie Database

```yaml
# DATABASE SERVER
- hosts: db
  become: true
  vars_files: vars/main.yml
  vars:
    root_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...

  tasks:
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

    - name: ensure database service is started (Debian family)
      service:
        name: mysql
        state: started
        enabled: yes
      when: ansible_facts['os_family'] == "Debian"

    - name: ensure database service is started (RedHat family)
      service:
        name: mariadb
        state: started
        enabled: yes
      when: ansible_facts['os_family'] == "RedHat"

    - name: Create MySQL client config
      copy:
        dest: "/root/.my.cnf"
        content: |
          [client]
          user=root
          password={{ root_password }}
        mode: 0400

    - name: Allow external MySQL connections (1/2) - Debian
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^skip-external-locking'
        line: "# skip-external-locking"
      when: ansible_facts['os_family'] == "Debian"
      notify: Restart database service

    - name: Allow external MySQL connections (2/2) - Debian
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^bind-address'
        line: "# bind-address"
      when: ansible_facts['os_family'] == "Debian"
      notify: Restart database service

    - name: Allow external MySQL connections - RedHat
      lineinfile:
        path: /etc/my.cnf
        regexp: '^bind-address'
        line: "bind-address = 0.0.0.0"
        insertafter: '^\[mysqld\]'
      when: ansible_facts['os_family'] == "RedHat"
      notify: Restart database service

    - name: upload sql table config
      template:
        src: "table.sql.j2"
        dest: "/tmp/table.sql"

    - name: add sql table to database
      mysql_db:
        name: "{{ mysql_dbname }}"
        state: present
        login_user: root
        login_password: "{{ root_password }}"
        state: import
        target: /tmp/table.sql

    - name: "Create {{ mysql_user }} with all {{ mysql_dbname }} privileges"
      mysql_user:
        name: "{{ mysql_user }}"
        password: "{{ mysql_password }}"
        priv: "{{ mysql_dbname }}.*:ALL"
        host: "{{ webserver_host }}"
        state: present
        login_user: root
        login_password: "{{ root_password }}"
        login_unix_socket: "{{ '/var/run/mysqld/mysqld.sock' if ansible_facts['os_family'] == 'Debian' else '/var/lib/mysql/mysql.sock' }}"

  handlers:
    - name: Restart database service
      service:
        name: "{{ 'mysql' if ansible_facts['os_family'] == 'Debian' else 'mariadb' }}"
        state: restarted
```

## Amélioration avec les variables conditionnelles

Pour éviter de répéter les conditions dans chaque tâche, nous pouvons définir des variables conditionnelles au début du play :

```yaml
---
- hosts: web
  become: true
  vars_files: vars/main.yml
  vars:
    # Variables conditionnelles basées sur la famille d'OS
    apache_package: "{{ 'apache2' if ansible_facts['os_family'] == 'Debian' else 'httpd' }}"
    apache_service: "{{ 'apache2' if ansible_facts['os_family'] == 'Debian' else 'httpd' }}"
    php_mysql_package: "{{ 'php-mysql' if ansible_facts['os_family'] == 'Debian' else 'php-mysqlnd' }}"
    
  tasks:
    - name: install apache and php (multi-distribution)
      package:
        name:
          - "{{ apache_package }}"
          - php
          - "{{ php_mysql_package }}"
        state: present

    - name: ensure apache service is started
      service:
        name: "{{ apache_service }}"
        state: started
        enabled: yes
```

:::info Module 'package' générique
Le module `package` est un module générique qui détecte automatiquement le gestionnaire de paquets à utiliser (`apt`, `yum`, `dnf`, etc.) selon la distribution. C'est une alternative pratique pour simplifier vos playbooks multi-distribution.
:::

## Test et validation du playbook multi-distribution

### Simulation sans machines RedHat

Même si vous ne disposez pas de machines RedHat, vous pouvez valider la syntaxe de votre playbook :

**1. Vérification de la syntaxe :**
```bash
ansible-playbook playbook.yml --syntax-check
```

**2. Mode simulation (dry-run) :**
```bash
ansible-playbook playbook.yml --check
```

**3. Limitation à un groupe spécifique :**
```bash
ansible-playbook playbook.yml --limit web
```

### Tester avec des machines virtuelles

Si vous souhaitez tester réellement le playbook multi-distribution, voici quelques options :

**Option 1 - Vagrant :**
Créez un `Vagrantfile` avec différentes distributions :

```ruby
Vagrant.configure("2") do |config|
  # Serveur web Ubuntu
  config.vm.define "web-ubuntu" do |web|
    web.vm.box = "ubuntu/focal64"
    web.vm.network "private_network", ip: "192.168.0.21"
  end

  # Serveur web CentOS
  config.vm.define "web-centos" do |web|
    web.vm.box = "centos/8"
    web.vm.network "private_network", ip: "192.168.0.23"
  end

  # Serveur DB Ubuntu
  config.vm.define "db-ubuntu" do |db|
    db.vm.box = "ubuntu/focal64"
    db.vm.network "private_network", ip: "192.168.0.22"
  end
end
```

**Option 2 - Conteneurs Docker :**
Utilisez des conteneurs avec systemd pour tester vos playbooks :

```bash
# Ubuntu
docker run -d --name ubuntu-test --privileged ubuntu/systemd

# CentOS
docker run -d --name centos-test --privileged centos:8
```

**Option 3 - Cloud providers :**
Utilisez des instances temporaires sur AWS, Azure, GCP, ou DigitalOcean.

### Inventaire pour tests multi-distribution

Adaptez votre inventaire pour inclure différentes distributions :

```yaml
all:
  children:
    web:
      hosts:
        web-ubuntu:
          ansible_host: 192.168.0.21
        web-centos:
          ansible_host: 192.168.0.23
    db:
      hosts:
        db-ubuntu:
          ansible_host: 192.168.0.22
        db-centos:
          ansible_host: 192.168.0.24
  vars:
    ansible_user: root
```

### Validation après exécution

**1. Vérifier les services :**
```bash
# Sur Debian/Ubuntu
ansible web -m shell -a "systemctl status apache2"
ansible db -m shell -a "systemctl status mysql"

# Sur RedHat/CentOS
ansible web -m shell -a "systemctl status httpd"
ansible db -m shell -a "systemctl status mariadb"
```

**2. Tester l'application web :**
```bash
curl http://192.168.0.21  # Ubuntu
curl http://192.168.0.23  # CentOS
```

**3. Vérifier la connectivité base de données :**
```bash
ansible web -m shell -a "php -r \"new PDO('mysql:host=192.168.0.22;dbname=blog', 'admin', 'secret');\""
```

## Bonnes pratiques pour les playbooks multi-distribution

### 1. Utiliser des variables de groupe

Créez des fichiers de variables spécifiques par famille d'OS :

**Structure recommandée :**
```
group_vars/
├── all.yml                    # Variables communes à tous
├── Debian.yml                 # Variables spécifiques à Debian
└── RedHat.yml                 # Variables spécifiques à RedHat
```

**group_vars/all.yml :**
```yaml
mysql_dbname: "blog"
mysql_user: "admin"
mysql_password: !vault |
  ...
```

**group_vars/Debian.yml :**
```yaml
apache_package: apache2
apache_service: apache2
mysql_package: mysql-server
mysql_service: mysql
mysql_socket: /var/run/mysqld/mysqld.sock
```

**group_vars/RedHat.yml :**
```yaml
apache_package: httpd
apache_service: httpd
mysql_package: mariadb-server
mysql_service: mariadb
mysql_socket: /var/lib/mysql/mysql.sock
```

### 2. Utiliser le module 'package' générique

Quand c'est possible, utilisez le module `package` au lieu de `apt` ou `yum` :

```yaml
- name: install apache
  package:
    name: "{{ apache_package }}"
    state: present
```

### 3. Créer des rôles Ansible

Pour des playbooks complexes, organisez votre code en rôles :

```
roles/
├── webserver/
│   ├── tasks/
│   │   ├── main.yml
│   │   ├── Debian.yml
│   │   └── RedHat.yml
│   ├── vars/
│   │   ├── Debian.yml
│   │   └── RedHat.yml
│   └── templates/
└── database/
    ├── tasks/
    ├── vars/
    └── templates/
```

### 4. Tester sur toutes les distributions cibles

Intégrez des tests automatisés dans votre pipeline CI/CD :

```yaml
# .gitlab-ci.yml ou .github/workflows/test.yml
test_debian:
  script:
    - ansible-playbook playbook.yml -i inventory-debian.yml --check

test_redhat:
  script:
    - ansible-playbook playbook.yml -i inventory-redhat.yml --check
```

### 5. Documenter les distributions supportées

Dans votre README ou documentation :

```markdown
## Distributions supportées

- ✅ Ubuntu 20.04, 22.04
- ✅ Debian 10, 11
- ✅ CentOS 8, 9
- ✅ Rocky Linux 8, 9
- ✅ AlmaLinux 8, 9
- ⚠️ Fedora (en cours de test)
- ❌ Arch Linux (non supporté)
```

## Conclusion

Dans ce tutoriel, nous avons appris à rendre notre playbook Ansible compatible avec plusieurs distributions Linux en utilisant :

**1. Les Facts Ansible :**
- Collecte automatique d'informations système
- Variable clé : `ansible_os_family`
- Filtrage avec le module `setup`

**2. Les conditions :**
- Instruction `when` pour exécuter des tâches conditionnellement
- Opérateurs de comparaison et logiques
- Tests avec le module `fail`

**3. Adaptations spécifiques :**
- Gestionnaires de paquets (`apt` vs `yum`)
- Noms de packages et services différents
- Chemins de configuration variables
- Gestion de SELinux sur RedHat

**4. Bonnes pratiques :**
- Variables conditionnelles pour éviter la répétition
- Module `package` générique
- Organisation en rôles
- Documentation des distributions supportées

### Avantages de l'approche multi-distribution

✅ **Flexibilité** : Un seul playbook pour gérer différents environnements  
✅ **Maintenabilité** : Modifications centralisées  
✅ **Réutilisabilité** : Partage facile entre projets et équipes  
✅ **Évolutivité** : Ajout simple de nouvelles distributions  
✅ **Conformité** : Respect des spécificités de chaque distribution

### Prochaines étapes

Pour aller plus loin, vous pouvez :

1. **Explorer les rôles Ansible** pour mieux organiser votre code
2. **Utiliser Ansible Galaxy** pour trouver des rôles communautaires multi-distribution
3. **Implémenter des tests avec Molecule** pour valider vos playbooks sur différentes distributions
4. **Apprendre Ansible Vault** pour sécuriser vos données sensibles (mots de passe, clés API)
5. **Découvrir les Collections Ansible** pour accéder à plus de modules et fonctionnalités

:::info Ressources complémentaires
- [Documentation officielle Ansible](https://docs.ansible.com/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Ansible Galaxy](https://galaxy.ansible.com/) : dépôt de rôles communautaires
- [Molecule](https://molecule.readthedocs.io/) : framework de test pour Ansible
:::

Vous disposez maintenant des connaissances nécessaires pour créer des playbooks Ansible flexibles et adaptables à différentes distributions Linux. Bonne automatisation ! 🚀
