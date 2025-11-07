# Création de notre playbook Ansible (stack LAMP)

## Introduction

Nous avions étudié dans le chapitre précédent comment lancer des modules en utilisant seulement la cli ansible. Dans ce chapitre, nous découvrirons un autre moyen d'exploiter les modules ansible à travers un fichier qu'on nomme le Playbook.

## Pourquoi les Playbooks

Par rapport aux modules utilisés précédemment exclusivement depuis la cli Ansible, les Playbooks sont utilisés dans des scénarios complexes et offrent une flexibilité accrue très bien adaptée au déploiement d'applications complexes. De plus les Playbooks sont plus susceptibles d'être gardés sous une source de contrôle (git) et d'assurer des configurations conformes aux spécifications de votre entreprise.

Les Playbooks sont exprimés au format YAML et ont un minimum de syntaxe, qui essaie intentionnellement de ne pas être un langage de programmation ou un script, mais plutôt un modèle de configuration (même s'il reste possible d'y intégrer des boucles et des conditions).

Comme ça utilise la syntaxe yaml, il faut faire attention à bien respecter l'indentation (c'est 2 espaces et non une tabulation pour faire une indentation).

Les Playbooks contiennent des tâches qui sont exécutées séquentiellement par l'utilisateur sur une machine particulière (ou un groupe de machine). Une tâche (Task) n'est rien de plus qu'un appel à un module ansible.

Dans ce chapitre nous allons créer une stack LAMP à partir d'un Playbook Ansible. Ce mini-projet va nous permettre d'examiner un exemple d'arborescence d'un projet Ansible et de découvrir quelques modules intéressants d'Ansible.

Récupérerez d'abord le projet complet [ici](../sources) et ensuite sans plus attendre, commençons par les explications !

## Structure du projet

### Quels sont les objectifs de notre Playbook ?

Ce Playbook Ansible nous fournira une alternative à l'exécution manuelle de la procédure d'installation générale d'un serveur LAMP (Linux, Apache, MySQL et PHP). L'exécution de ce Playbook automatisera donc les actions suivantes sur nos hôtes distants :

**Côté serveur web :**

- Installer les packages apache2, php et php-mysql
- Déployer les sources de notre application dans notre serveur web distant
- S'assurer que le service apache est bien démarré

**Côté serveur base de données :**

- Installer les packages mysql
- Modifier le mot de passe root
- Autoriser notre serveur web à communiquer avec la base de données
- Configurer notre table mysql avec les bonnes colonnes et autorisations

### Arborescence du projet

Voici à quoi ressemble l'arborescence de note projet une fois téléchargé :

```
|── files
│   └── app
│       ├── index.php
│       └── validation.php
|
|── templates
│   ├── db-config.php.j2
│   └── table.sql.j2
|
|── vars
│   └── main.yml
|
└── ansible.cfg
└── inventory.yml
└── playbook.yml
```

- **playbook.yml** : fichier Playbook contenant les tâches à exécuter sur le ou les serveurs distants.
- **vars/main.yml** : fichier pour nos variables afin de personnaliser les paramètres du Playbook (on peut aussi déclarer des variables dans le fichier Playbook).
- **inventory.yml** : Fichier inventaire de notre Playbook au format YAML.
- **ansible.cfg** : par défaut ansible utilise le fichier de configuration /etc/ansible/ansible.cfg mais on peut surcharger la config en rajoutant un fichier nommé ansible.cfg à la racine du projet.
- **files/** : contient les sources de notre stack LAMP qui seront par la suite destinés à être traités par le module [copy](https://docs.ansible.com/ansible/latest/modules/copy_module.html).
- **templates/** : contient des modèles de configurations dynamiques au format [jinja](https://jinja.palletsprojects.com/en/2.10.x/) qui sont destinés à être traités par le module [template](https://docs.ansible.com/ansible/latest/modules/template_module.html).

## Le fichier inventaire

Pour ce projet, nous avons décidé de nous séparer du fichier inventaire situé par défaut dans /etc/ansible/hosts et de créer notre propre fichier d'inventaire à la racine du projet. Nous allons utiliser le format YAML qui offre une structure plus claire et plus flexible que le format INI traditionnel.

Nous séparons le serveur de base de données du serveur web en créant deux groupes distincts : `web` et `db`. Voici à quoi ressemble notre fichier inventaire `inventory.yml` :

```yaml
all:
  children:
    web:
      hosts:
        node-web:
          ansible_host: 192.168.0.21
    db:
      hosts:
        node-db:
          ansible_host: 192.168.0.22
  vars:
    ansible_user: root
```

**Structure de l'inventaire :**

- **all** : groupe parent qui contient tous les hôtes
- **children** : permet de définir des sous-groupes (web et db dans notre cas)
- **web** : groupe contenant le(s) serveur(s) web
- **db** : groupe contenant le(s) serveur(s) de base de données
- **vars** : variables globales applicables à tous les hôtes (ici l'utilisateur par défaut est `root`)

:::info Information
Vous pouvez adapter cet inventaire selon vos besoins :
- Ajouter plusieurs nodes dans chaque groupe
- Modifier les adresses IP (`ansible_host`) selon votre infrastructure
- Ajouter des variables spécifiques par groupe ou par hôte
- Utiliser des noms d'hôtes DNS au lieu d'adresses IP

Exemple avec plusieurs nodes :
```yaml
all:
  children:
    web:
      hosts:
        node-web-01:
          ansible_host: 192.168.0.21
        node-web-02:
          ansible_host: 192.168.0.22
    db:
      hosts:
        node-db-01:
          ansible_host: 192.168.0.31
  vars:
    ansible_user: root
```
:::

Pour que notre nouveau fichier inventaire personnalisé soit pris en compte par votre Playbook, il faut au préalable modifier la valeur de la variable `inventory` située dans notre fichier de configuration ansible.

Par défaut ce fichier se situe dans /etc/ansible/ansible.cfg. Mais pour faire les choses dans les règles de l'art, nous allons laisser la configuration par défaut choisie par Ansible et créer notre propre fichier de configuration à la racine du projet. Dans notre nouveau fichier de config nous surchargerons uniquement la valeur de la variable `inventory`, ce qui nous donne le fichier `ansible.cfg` suivant :

```ini
[defaults]
inventory = ./inventory.yml
```

Ce fichier de configuration indique à Ansible d'utiliser notre fichier `inventory.yml` situé à la racine du projet au lieu de l'inventaire par défaut.

## Explication du Playbook

Pour commencer, voici le contenu de notre Playbook de départ :

```yaml
---

# WEB SERVER
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
    # Create the web server document root directory
    # Sets up /var/www/html with proper permissions (755) for web content
    # This ensures the directory exists and is accessible by the web server
    # ansible.builtin.file:
      # path: /var/www/html
      # ...

  - name: remove default index.html
    # Remove the default index.html file from the web server document root
    # This task must delete /var/www/html/index.html to prepare for custom content deployment
    # ansible.builtin.file:
      # ...

  - name: upload web app source
    # Copy application files from local app/ directory to web server document root
    # so copy app/ to /var/www/html/
    # ansible.builtin.copy:
      # src: ...
      # dest: ...

  - name: deploy php database config
    # Deploy PHP database configuration file from ansible jinja template
    # This creates db-config.php in the web root with database connection parameters
    # The template will be populated with variables like mysql_dbname, mysql_user, etc.
    # ansible.builtin.template:
      # src: ... my local jinja config file
      # dest: "/var/www/html/db-config.php"
  
  - name: ensure apache service is start
    # Start and enable the Apache2 web server service
    # This task ensures that:
    # - Apache2 service is started (running)
    # - Apache2 service is enabled to start automatically on system boot
    # ansible.builtin.service:
      # name: ...
      # state: ...
      # ...


# DATABASE SERVER

- hosts: db
  become: true
  vars_files: vars/main.yml
  vars:
    root_password: "my_secret_password"

  tasks:
  - name: install mysql
    apt:
      name: 
        - mysql-server
        - python-mysqldb # for mysql_db and mysql_user modules
      state: present
      update_cache: yes  

  - name: Create MySQL client config
    copy:
      dest: "/root/.my.cnf"
      content: |
        [client]
        user=root
        password={{ root_password }}
      mode: 0400

  - name: Allow external MySQL connections (1/2)
    lineinfile:
      path: /etc/mysql/mysql.conf.d/mysqld.cnf
      regexp: '^skip-external-locking'
      line: "# skip-external-locking"
    notify: Restart mysql

  - name: Allow external MySQL connections (2/2)
    lineinfile:
      path: /etc/mysql/mysql.conf.d/mysqld.cnf
      regexp: '^bind-address'
      line: "# bind-address"
    notify: Restart mysql

  - name: upload sql table config
    template:
      src: "table.sql.j2"
      dest: "/tmp/table.sql"

  - name: add sql table to database
    mysql_db:
      name: "{{ mysql_dbname }}"
      state: present
      login_user: root
      login_password: '{{ root_password }}'
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
      login_password: '{{ root_password }}'
      login_unix_socket: /var/run/mysqld/mysqld.sock

  handlers:
    - name: Restart mysql
      service:
        name: mysql
        state: restarted
```

:::info Note importante
Ce playbook contient des tâches à compléter (marquées par des commentaires `# ...`). Dans la partie serveur web, vous devrez :
- Configurer le module `file` pour créer le répertoire avec les bonnes permissions
- Supprimer le fichier index.html par défaut
- Utiliser le module `copy` pour déployer les sources de l'application
- Utiliser le module `template` pour déployer la configuration de base de données
- Configurer le module `service` pour démarrer Apache

Les tâches de la partie base de données sont déjà complètes et serviront d'exemples. Suivez les instructions détaillées dans les sections suivantes pour compléter ces tâches.
:::

Comme dit précédemment, nous avons choisi de séparer dans notre nouveau fichier inventaire le serveur de base de données par rapport à notre serveur web. Notre Playbook doit continuer dans cette voie en ciblant d'abord le serveur Web, puis le serveur de base de données (ou inversement). Vous pouvez compléter le playbook en suivant le déroulé dans les parties suivantes.

## Serveur web

Dans cette partie, nous nous intéresserons particulièrement à la partie Web de notre playbook.

### Partie hosts

Pour chaque jeu dans un Playbook, vous pouvez choisir les machines à cibler pour effectuer vos tâches. Dans notre cas on commence par cibler notre serveur web :

```yaml
---
- hosts: web
```

:::info Information
Les 3 tirets au début d'un fichier yaml ne sont pas obligatoires.
:::

### Élévation de privilèges

On demande au moteur Ansible d'exécuter toutes nos tâches en tant qu'utilisateur root grâce au mot-clé become :

```yaml
become: true
```

Vous pouvez également utiliser le mot-clé become sur une tâche particulière au lieu de l'ensemble de vos tâches :

```yaml
tasks:
  - service:
      name: nginx
      state: started
    become: yes
```

### Variables

Concernant les variables, vous avez le choix entre les placer directement depuis le mot-clé vars, ou vous pouvez les charger depuis un fichier en utilisant le mot-clé vars_files comme ceci :

```yaml
vars_files: vars/main.yml
```

Voici le contenu du fichier de variables (nous verrons par la suite comment sécuriser ce type de variables / infos sensibles avec Ansible Vault) :

```yaml
---
mysql_user: "admin"
mysql_password: "secret"
mysql_dbname: "blog"
db_host: "192.168.0.22"
webserver_host: "192.168.0.21"
```

- **mysql_user** : l'utilisateur de notre base de données mysql qui exécutera nos requêtes SQL depuis notre application web.
- **mysql_password** : le mot de passe de l'utilisateur de notre base de données mysql.
- **mysql_dbname** : le nom de notre base de données.
- **db_host** : l'adresse IP de notre serveur mysql (utilisée dans la configuration de connexion de l'application web). Cette valeur doit correspondre à celle définie dans l'inventaire pour le serveur de base de données.
- **webserver_host** : l'adresse IP du serveur web (utilisée pour autoriser uniquement l'ip du serveur web à communiquer avec notre base de données). Cette valeur doit correspondre à celle définie dans l'inventaire pour le serveur web.

:::info Information
Les adresses IP définies ici (`db_host` et `webserver_host`) sont utilisées dans la configuration applicative et les règles d'accès MySQL. Elles doivent correspondre aux adresses IP définies dans votre fichier `inventory.yml` pour éviter tout problème de connectivité.
:::

### Les tâches

Chaque hôte contient une liste de tâches au-dessous du mot-clé tasks. Les tâches sont exécutées dans l'ordre, une à la fois, sur toutes les machines correspondant au modèle d'hôte avant de passer à la tâche suivante.

Le but de chaque tâche est d'exécuter un module Ansible avec des arguments très spécifiques. Les variables peuvent également être utilisées dans les arguments des modules.

Chaque tâche peut débuter avec le mot-clé name, qui est simplement une briefe description de votre tâche. Cette information s'affichera à la sortie de l'exécution du Playbook, son but principal est de pouvoir distinguer et décrire vos différentes tâches. Il est donc utile de fournir de bonnes petites descriptions pour chaque tâche. Si le champ n'est pas saisi alors le nom du module sera utilisée comme sorties. Au-dessous du mot-clé name, vous insérez le nom du module avec ses différents paramètres.

Dans notre projet, notre premier besoin consiste à installer les packages apache2, php et php-mysql avec le gestionnaire de paquêts apt.

Et peut-être que vous vous demandez comment trouver le module adéquat ? La réponse est "Google !", en effet Google est votre meilleur ami (ou Bing, Ecosia, Qwant, DuckDuckGo, etc ...) !

Nous pouvons taper sur le moteur de recherche les mots-clés suivants "Ansible apt module" et cliquer sur le premier lien fourni par Google ([celui-ci](https://docs.ansible.com/ansible/latest/modules/apt_module.html)).

![Comment rechercher un module ansible sur le moteur de recherches Google](https://devopssec.fr/images/articles/ansible/playbooks/how_to_search_ansible_module_on_google.jpg)

Sur cette page vous avez le Synopsis qui vous fournit une description courte du module :

![ansible synopsis du module apt](https://devopssec.fr/images/articles/ansible/playbooks/ansible_synopsis.jpg)

Si on traduit mot par mot le Synopsis, nous aurons la phrase suivante : "Gère les paquets apt (comme pour Debian/Ubuntu)".

Ça correspond parfaitement à notre besoin ! Maintenant l'étape suivante consiste à rechercher les différents paramètres que propose le module apt. Dans notre cas on cherche à installer la dernière version des packages apache2, php et php-mysql. En lisant la documentation on peut vite s'apercevoir qu'il existe les options suivantes :

- **name** (type: liste) : liste de noms de packages (on peut aussi spécifier la version du package ex curl=1.6 ou curl=1.0*).
- **state** (type: string) : indique l'état du package, voici un exemple des valeurs possibles :
  - **latests** : assure que c'est toujours la dernière version qui est installée.
  - **present** : vérifie si le package est déjà installé, si c'est le cas il ne fait rien, sinon il l'installe.
  - **absent** : supprime le package s'il est déjà installé.
- **update_cache** (type: booléen) : exécute l'équivalent de la commande apt-get update avant l'installation des paquets.

Si on combine toutes ces informations on se retrouve avec la tâche suivante :

```yaml
- name: install apache and php last version
  apt:
    name:
      - apache2
      - php
      - php-mysql
    state: present
    update_cache: yes
```

Nous avons utilisé la même méthodologie de recherche pour retrouver le reste des tâches de ce Playbook.

#### Les types en Yaml :

Prenons quelques instants pour expliquer l'utilisation de quelques types de variables dans le langage Yaml. En effet, vous avez différentes façons pour valoriser vos variables selon leurs types.

Par exemple, pour le paramètre name du module apt qui est de type list, on peut aussi l'écrire comme une liste sur python, soit :

```yaml
- name: install apache and php last version
  apt:
    name: ['apache2', 'php', 'php-mysql']
    state: present
    update_cache: yes
```

Concernant les types booléens, comme pour le paramètre update_cache, vous pouvez spécifier une valeur sous plusieurs formes:

```yaml
update_cache: yes
update_cache: no
update_cache: True
update_cache: TRUE
update_cache: false
```

Vous avez aussi la possibilité de raccourcir la tâche d'un module. Prenons par exemple la tâche suivante :

```yaml
tasks:
  - name: deploy test.cfg file
    copy:
      src: /tmp/test.cfg
      dest: /tmp/test.cfg
      owner: root
      group: root
      mode: 0644
```

Pour la raccourcir, il suffit de mettre tous vos paramètres sur une seule ligne (possibilité de faire un saut à la ligne) et de remplacer les : par des =. Ce qui nous donne :

```yaml
tasks:
  - name: deploy test.cfg file
    copy: src=/tmp/test.cfg dest=/tmp/test.cfg
          owner=root group=root mode=0644
```

### Idempotence

Les modules doivent être idempotents, c'est-à-dire que l'exécution d'un module plusieurs fois dans une séquence doit avoir le même effet que son exécution unique.

Les modules fournis par Ansible sont en général idempotents, mais il se peut que vous ne trouveriez pas des modules répondant parfaitement à votre besoin, dans ce cas vous passerez probablement par le module [command](https://docs.ansible.com/ansible/latest/modules/command_module.html) ou [shell](https://docs.ansible.com/ansible/latest/modules/shell_module.html) qui vont vous permettre ainsi d'exécuter vos propres commandes shell.

Si vous êtes amené à travailler avec ces modules dans votre Playbook, il faut faire attention à ce que vos tâches soient idempotentes, la réexécution du Playbook doit être sûre.

Cette parenthèse étant fermée, on peut continuer par l'explication de notre Playbook

### Suite des tâches

~~Installer les packages apache2, php et php-mysql~~

- Déployer les sources de notre application dans notre serveur web distant
- S'assurer que le service apache est bien démarré

Pour déployer les sources de notre application, il faut au préalable donner les droits d'écriture sur le dossier /var/www/html, pour cela rien de mieux que d'utiliser le module file ([documentation ici](https://docs.ansible.com/ansible/latest/modules/file_module.html)) qui permet entre autres de gérer les propriétés des fichiers/dossiers.

```yaml
- name: Give writable mode to http folder
  file:
    path: /var/www/html
    state: directory
    mode: '0755'
```

Nous enchaînons ensuite par la suppression de la page d'accueil du serveur apache, en éliminant le fichier index.html.

```yaml
- name: remove default index.html
  file:
    path: /var/www/html/index.html
    state: absent
```

Une fois que nous avons les droits d'écriture dans ce dossier, la prochaine étape comprend l'upload des sources de notre application dans le dossier /var/www/html de notre serveur web distant.

Un des modules qui peut répondre à une partie de notre besoin, est le module copy ([Documentation ici](https://docs.ansible.com/ansible/latest/modules/copy_module.html)) qui permet de copier des fichiers ou des dossiers de notre serveur de contrôl vers des emplacements distants.

```yaml
- name: upload web app source
  copy:
    src: app/
    dest: /var/www/html/
```

Peut-être que vous l'avez remarqué, mais nous n'avons pas besoin de fournir le dossier files dans le chemin du paramètre src, car ce dossier est spécialement conçu pour que le module copy recherche dedans automatiquement nos différents fichiers ou dossiers à envoyer (si vous déposez vos fichiers dans un autre emplacement, il faut dans ce cas que vous insériez le chemin relatif ou absolu complet)

### Fichier de configuration dynamique (Jinja2)

Cependant, nous allons être confrontés à un problème. En effet, nous avons déclaré des variables dans le fichier vars/main.yml, dont quelques-unes pour se connecter à notre base de données. Comme par exemple l'utilisateur et le mot de passe mysql.

Il nous faut donc un moyen pour que notre fichier php, qui permet la connexion à la base données, soit automatiquement en accord avec ce que l'utilisateur a décidé de valoriser dans le fichier vars/main.yml.

La solution à ce problème est l'utilisation du module template ([Documentation ici](https://docs.ansible.com/ansible/latest/modules/template_module.html)).

Il permet de faire la même chose que le module copy. Cependant, ce module permet de modifier dynamiquement un fichier avant de l'envoyer sur le serveur cible.

Pour ce faire les fichiers sont écrits et traités par le [langage Jinja2](https://jinja.palletsprojects.com/en/2.10.x/).

Nous n'entrerons pas trop dans les détails de ce langage, mais concernant notre besoin, où il s'agit de remplacer certaines valeurs de notre fichier php, on exploitera les variables dans le langage Jinja2.

Vous pouvez effectivement, jouer avec les variables dans les modèles jinja qui seront au préalable valorisées par le module template. Il suffit donc dans notre fichier jinja de reprendre le même nom que notre variable Ansible et de la mettre entre deux accolades, voici par exemple le contenu de notre template db-config.php.j2 :

```php
<?php
const DB_DSN = 'mysql:host={{ db_host }};dbname={{ mysql_dbname }}';
const DB_USER = "{{ mysql_user }}";
const DB_PASS = "{{ mysql_password }}";

$options = array(
    PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES utf8", // encodage utf-8
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION, // gérer les erreurs en tant qu'exception
    PDO::ATTR_EMULATE_PREPARES => false // faire des vrais requêtes préparées et non une émulation
);
```

Par exemple, pour ce même fichier, le module template remplacera {{ mysql_user }} par la valeur de la variable mysql_user située dans le fichier vars/main.yml avant de l'envoyer sur notre serveur web.

Ce qui nous donne la tâche suivante :

```yaml
- name: deploy php database config
  template:
    src: "db-config.php.j2"
    dest: "/var/www/html/db-config.php"
```

:::info Information
Comme pour le module copy, ici nul besoin de fournir le dossier templates/ dans le chemin du paramètre src, car le module template recherche automatiquement nos différents fichiers jinja dans ce dossier (si vous déposez vos fichiers dans un autre emplacement, il faut dans ce cas que vous insériez le chemin relatif ou absolu complet).
:::

### Le module service

~~Installer les packages apache2, php et php-mysql~~

~~Déployer les sources de notre application dans notre serveur web distant~~

- S'assurer que le service apache est bien démarré

Quand il s'agit de gérer des services Linux, il faut penser directement au module service ([Documentation ici](https://docs.ansible.com/ansible/latest/modules/service_module.html)).

Il reste très simple à utiliser, il suffit simplement de lui fournir le nom du service à gérer dans le paramètre name, ainsi que l'état souhaité du service dans le paramètre state, qui peut contenir les valeurs suivantes :

- **reloaded** : recharger le service sans le stopper
- **restarted** : redémarrage du service (arrêt + démarrage)
- **started** : si nécessaire le service sera démarré
- **stopped** : si nécessaire le service sera arrêté

Voici à quoi ressemble notre dernière tâche de notre serveur web :

```yaml
- name: ensure apache service is start
  service:
    name: apache2
    state: started
    enabled: yes
```

Voilà, dorénavant les tâches de notre serveur web sont finalisées. On s'attaquera maintenant à l'hôte de base de données.

## Serveur de base de données

Comme pour notre serveur web, on commence d'abord par préparer le terrain pour les différentes tâches de notre hôte de base de données.

### Préparation des tâches

Comme pour notre serveur web, nous utilisons le mot-clé become pour l'élévation de privilèges et le mot-clé vars_files pour inclure les variables situées dans le fichier vars/main.yml. Cependant, nous avons choisi de placer une variable uniquement utilisable par les tâches de notre serveur de base données, soit la variable root_password. Ce qui nous donne la configuration suivante :

```yaml
- hosts: db
  become: true
  vars_files: vars/main.yml
  vars:
    root_password: "my_secret_password"
```

### Sécurisation avec Ansible Vault

**Qu'est-ce qu'Ansible Vault ?**

Ansible Vault est un outil intégré à Ansible qui permet de chiffrer des données sensibles (mots de passe, clés API, certificats, etc.) afin de les stocker en toute sécurité dans vos fichiers de configuration, même dans un système de contrôle de version comme Git.

**Pourquoi utiliser Ansible Vault ici ?**

Dans notre exemple, la variable `root_password` contient un mot de passe en clair, ce qui représente un risque de sécurité. Avec Ansible Vault, nous pouvons chiffrer cette valeur pour la protéger.

**Étapes pour chiffrer une variable avec Ansible Vault :**

1. **Chiffrer la valeur du mot de passe** :

```bash
ansible-vault encrypt_string 'my_secret_password' --name 'root_password'
```

Cette commande vous demandera de créer un mot de passe vault (à retenir ou à stocker dans un gestionnaire de mots de passe). Elle générera ensuite une sortie chiffrée.

2. **Remplacer la valeur en clair dans le playbook** :

Au lieu de :
```yaml
vars:
  root_password: "my_secret_password"
```

Utilisez la valeur chiffrée :
```yaml
vars:
  root_password: !vault |
    $ANSIBLE_VAULT;1.1;AES256
    66386439653233653966663633353639643535343264626361653433646565646536363336306437
    3335616264396164343038636366633663356534393062630a316564663034383937373563623638
    62313235306631313331643965316362303038343564626536613432313562653561616635363832
    6464343038636138360a636137666162636565323762346262363137373734323131656538623733
    3764
```

3. **Exécuter le playbook avec le mot de passe vault** :

```bash
# Option 1 : Ansible demande le mot de passe interactivement
ansible-playbook playbook.yml --ask-vault-pass

# Option 2 : Utiliser un fichier contenant le mot de passe vault
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass.txt
```

:::info Note sur les IDE
Certains éditeurs de code (VS Code, PyCharm, etc.) peuvent souligner la syntaxe `!vault` comme une erreur ou un avertissement. C'est normal : il s'agit d'une syntaxe YAML spécifique à Ansible. Vous pouvez ignorer ces avertissements, votre playbook fonctionnera correctement.
:::

**Bonnes pratiques :**

- Ne commitez jamais le fichier contenant le mot de passe vault (`.vault_pass.txt`) dans Git
- Ajoutez `.vault_pass.txt` à votre `.gitignore`
- Partagez le mot de passe vault de manière sécurisée avec votre équipe (gestionnaire de mots de passe d'équipe)
- Vous pouvez chiffrer des variables individuelles (comme ici) ou des fichiers entiers avec `ansible-vault encrypt vars/main.yml`

### Installation des paquets

Pour rappel, voici les étapes à effectuer sur notre serveur de base de données :

- Installer les packages mysql
- Modifier le mot de passe root
- Autoriser notre serveur web à communiquer avec la base de données
- Configurer notre table mysql avec les bonnes colonnes et autorisations

```yaml
- name: install mysql
  apt:
    name:
      - mysql-server
      - python-mysqldb # for mysql_db and mysql_user modules
    state: present
    update_cache: yes
```

On utilise une nouvelle fois le module apt afin d'installer nos différents packages. Le package mysql-server nous permet d'installer notre base de données relationnelle. Ensuite on installe le package python-mysqldb qui est nécessaire pour utiliser plus tard le module [mysql_user](https://docs.ansible.com/ansible/latest/modules/mysql_user_module.html) and [mysql_db](https://docs.ansible.com/ansible/latest/modules/mysql_db_module.html).

### Modification du mot de passe root

Il existe différentes manières pour modifier le mot de passe mysql du compte root. Pour notre part, nous avons choisi de surcharger le fichier de configuration mysql par défaut. Pour cela, nous créons sur le serveur distant un fichier .my.cnf à l'emplacement /root/. Pour cela, nous utilisons le module copy, mais cette fois-ci avec le paramètre content à la place du paramètre src. Lorsque ce paramètre est utilisé à la place de src, on peut comme son nom l'indique définir le contenu d'un fichier directement sur la valeur spécifiée. Ce qui nous donne :

```yaml
- name: Create MySQL client config
  copy:
    dest: "/root/.my.cnf"
    content: |
      [client]
      user=root
      password={{ root_password }}
```

La valeur {{ root_password }} sera bien sûr remplacée par la valeur de variable root_password soit dans cet exemple la valeur "my_secret_password".

:::info Information
Pour créer un contenu multiligne il faut utiliser le caractère | après le nom du module, comme nous pouvons le faire pour cet exemple.
:::

### Autorisation des connexions externes

Pour autoriser les communications externes sur notre serveur mysql, on peut commenter la ligne commençant par bind-address et skip-external-locking dans le fichier de configuration /etc/mysql/mysql.conf.d/mysqld.cnf du serveur mysql distant.

Quand il s'agit de faire des modifications sur des fichiers distants, le module le plus adapté reste le module lineinfile ([Documentation ici](https://docs.ansible.com/ansible/latest/modules/lineinfile_module.html)).

C'est un module spécialement conçu pour gérer les lignes dans les fichiers texte. Dans notre cas il nous est demandé de commenter des lignes commençant par un mot bien particulier. Pour cela, nous aurons besoin des expressions régulières, soit le paramètre regexp du module lineinfile et le paramètre line pour la ligne de remplacement. Ce qui nous donne le résultat suivant :

```yaml
- name: Allow external MySQL connections (1/2)
  lineinfile:
    path: /etc/mysql/mysql.conf.d/mysqld.cnf
    regexp: '^skip-external-locking'
    line: "# skip-external-locking"
  notify: Restart mysql

- name: Allow external MySQL connections (2/2)
  lineinfile:
    path: /etc/mysql/mysql.conf.d/mysqld.cnf
    regexp: '^bind-address'
    line: "# bind-address"
  notify: Restart mysql
```

#### notify et handlers

Vous remarquerez que j'utilise le mot-clé notify (notification en français). Ce sont tout simplement des actions (tâches) qui sont déclenchées à la fin de chaque bloc de tâches.

Ces actions sont répertoriées dans la partie handlers. Les handlers sont des listes de tâches, qui ne diffèrent pas vraiment des tâches normales, qui sont référencées par un nom globalement unique et qui sont déclenchées par le mot-clé notify.

Dans notre cas c'est le handler suivant qui est déclenché à la fin de notre tâche :

```yaml
handlers:
  - name: Restart mysql
    service:
      name: mysql
      state: restarted
```

### Création et configuration de notre base de données

Notre serveur mysql est dorénavant démarré et configuré pour accepter des connexions externes.

La prochaine étape est de créer notre table et notre utilisateur mysql avec les privilèges appropriés.

Pour ce faire, nous avons besoin de deux modules : le module template pour adapter notre fichier sql (fichier qui contient la structure de notre base de données) avant de l'envoyer au serveur distant, qui sera par la suite exécuté par le module mysql_db ([Documentation ici](https://docs.ansible.com/ansible/latest/modules/mysql_db_module.html)) :

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
    login_password: '{{ root_password }}'
    state: import
    target: /tmp/table.sql
```

:::info Information
Bien sûr notre base de données sera créée grâce au paramètre name avant d'exécuter notre fichier sql défini sur le paramètre target (ce qui est assez logique sinon on se retrouvera avec des erreurs 😅)
:::

La dernière étape de configuration est de créer notre utilisateur mysql définit dans le fichier vars/main.yml , et de lui fournir les autorisations uniquement sur notre base de données fraîchement crée précédemment. Il ne faut pas oublier aussi d'autoriser uniquement notre serveur web à communiquer avec notre base de données.

Toutes ces exigences peuvent être résolues grâce au module mysql_user ([Documentation ici](https://docs.ansible.com/ansible/latest/modules/mysql_user_module.html)). Ce qui nous donne la tâche suivante :

```yaml
- name: "Create {{ mysql_user }} with all {{ mysql_dbname }} privileges"
  mysql_user:
    name: "{{ mysql_user }}"
    password: "{{ mysql_password }}"
    priv: "{{ mysql_dbname }}.*:ALL"
    host: "{{ webserver_host }}"
    state: present
    login_user: root
    login_password: '{{ root_password }}'
    login_unix_socket: /var/run/mysqld/mysqld.sock
```

## Test

Avant de lancer votre playbook, assurez-vous que :
1. Les adresses IP dans `inventory.yml` correspondent à vos serveurs réels
2. Les variables `db_host` et `webserver_host` dans `vars/main.yml` correspondent aux adresses IP de votre inventaire
3. Vous avez un accès SSH fonctionnel aux deux serveurs avec l'utilisateur root

Voici la commande pour lancer votre playbook :

```bash
ansible-playbook playbook.yml
```

Si tout s'est bien déroulé, visitez la page [http://IP_SERVEUR_WEB](http://IP_SERVEUR_WEB) (remplacez IP_SERVEUR_WEB par l'adresse IP de votre serveur web définie dans l'inventaire), et vous obtiendrez la page d'accueil suivante :

![page d'accueil du serveur web déployé depuis Ansible](https://devopssec.fr/images/articles/ansible/playbooks/home_page.jpg)

Pour tester la connexion à notre base de données, nous allons appuyer sur le bouton "Envoyer" pour valider le formulaire et rajouter un article à la base de données, ce qui nous donne le résultat suivant :

![page d'articles du serveur web déployé depuis Ansible](https://devopssec.fr/images/articles/ansible/playbooks/add_article.jpg)

## Conclusion

Je pense que vous l'aurez compris, le Playbook est un fichier permettant de faciliter la gestion de nos modules Ansible. Nous verrons dans le prochain chapitre comment améliorer notre playbook avec les conditions et nous aborderons également les boucles dans les playbooks.
