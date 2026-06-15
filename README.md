# Déploiement Apache avec Ansible

Voici une version propre adaptée à mon lab Ansible.

## Contexte du lab

```text
Machine de contrôle : toto@ansible
Client cible        : client-apache
IP client           : 172.16.93.172
User SSH distant    : ansible
Sudo                : sans mot de passe
SSH                 : par clé
Inventaire          : /home/toto/ansible/inventories/hosts.ini
```

Ce projet utilise uniquement des modules `ansible.builtin.*`, inclus dans `ansible-core`.

Modules utilisés :

```text
ansible.builtin.apt
ansible.builtin.file
ansible.builtin.template
ansible.builtin.service
ansible.builtin.ping
```

Documentation officielle Ansible :

```text
https://docs.ansible.com/projects/ansible/latest/index.html
```

---

## 1. Arborescence recommandée

Dans `/home/toto/ansible`, garder cette structure :

```text
/home/toto/ansible/
├── ansible.cfg
├── inventories/
│   └── hosts.ini
├── playbooks/
│   └── apache.yml
└── roles/
    └── apache/
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        └── templates/
            ├── apache-site.conf.j2
            └── index.html.j2
```

Commande pour créer les dossiers :

```bash
mkdir -p /home/toto/ansible/{inventories,playbooks}
mkdir -p /home/toto/ansible/roles/apache/{defaults,handlers,tasks,templates}
```

Commande pour créer les fichiers :

```bash
touch /home/toto/ansible/ansible.cfg
touch /home/toto/ansible/inventories/hosts.ini
touch /home/toto/ansible/playbooks/apache.yml
touch /home/toto/ansible/roles/apache/defaults/main.yml
touch /home/toto/ansible/roles/apache/tasks/main.yml
touch /home/toto/ansible/roles/apache/handlers/main.yml
touch /home/toto/ansible/roles/apache/templates/apache-site.conf.j2
touch /home/toto/ansible/roles/apache/templates/index.html.j2
```

Les rôles Ansible chargent automatiquement les fichiers `tasks`, `handlers`, `defaults`, `templates`, etc., selon l’arborescence officielle des rôles.

---

## 2. Fichier `ansible.cfg`

Chemin :

```text
/home/toto/ansible/ansible.cfg
```

Contenu :

```ini
[defaults]
inventory = ./inventories/hosts.ini
roles_path = ./roles
host_key_checking = False
interpreter_python = auto_silent

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

Comme le fichier `ansible.cfg` est placé dans `/home/toto/ansible`, le chemin relatif suivant :

```text
./inventories/hosts.ini
```

pointe vers :

```text
/home/toto/ansible/inventories/hosts.ini
```

Comme l’utilisateur distant `ansible` possède déjà les droits sudo sans mot de passe, la directive suivante est correcte :

```ini
become_ask_pass = False
```

---

## 3. Inventaire `hosts.ini`

Chemin :

```text
/home/toto/ansible/inventories/hosts.ini
```

Contenu :

```ini
[webservers]
client-apache ansible_host=172.16.93.172 ansible_user=ansible ansible_port=22

[webservers:vars]
ansible_become=true
ansible_become_method=sudo
```

L’inventaire définit les hôtes gérés, les groupes et les variables associées aux hôtes.

Ici, le client est placé dans le groupe :

```text
webservers
```

L’alias Ansible du client est :

```text
client-apache
```

L’adresse réelle du client est définie avec :

```ini
ansible_host=172.16.93.172
```

---

## 4. Playbook `apache.yml`

Chemin :

```text
/home/toto/ansible/playbooks/apache.yml
```

Contenu :

```yaml
---
- name: Déployer Apache sur le client Ansible
  hosts: webservers
  become: true

  roles:
    - apache
```

Ce playbook cible le groupe `webservers` défini dans l’inventaire et appelle le rôle `apache`.

---

## 5. Rôle `apache`

### 5.1 Fichier `defaults/main.yml`

Chemin :

```text
/home/toto/ansible/roles/apache/defaults/main.yml
```

Contenu :

```yaml
---
apache_package_name: apache2
apache_service_name: apache2

apache_listen_port: 80
apache_server_name: "_"

apache_document_root: /var/www/html
apache_index_file: /var/www/html/index.html

apache_site_available: /etc/apache2/sites-available/000-default.conf
```

Ce fichier contient les variables par défaut du rôle `apache`.

---

### 5.2 Fichier `tasks/main.yml`

Chemin :

```text
/home/toto/ansible/roles/apache/tasks/main.yml
```

Contenu :

```yaml
---
- name: Mettre à jour le cache APT
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600

- name: Installer Apache
  ansible.builtin.apt:
    name: "{{ apache_package_name }}"
    state: present

- name: Vérifier que le dossier web existe
  ansible.builtin.file:
    path: "{{ apache_document_root }}"
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: Déployer la configuration du site Apache par défaut
  ansible.builtin.template:
    src: apache-site.conf.j2
    dest: "{{ apache_site_available }}"
    owner: root
    group: root
    mode: "0644"
  notify: Reload apache

- name: Déployer la page d'accueil Apache
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ apache_index_file }}"
    owner: root
    group: root
    mode: "0644"
  notify: Reload apache

- name: Démarrer et activer le service Apache
  ansible.builtin.service:
    name: "{{ apache_service_name }}"
    state: started
    enabled: true
```

Le module `ansible.builtin.apt` gère les paquets APT. Il est donc adapté pour installer `apache2` sur une distribution Debian ou Ubuntu.

Le module `ansible.builtin.file` permet de gérer les fichiers et dossiers.

Le module `ansible.builtin.template` génère un fichier distant depuis un template Jinja2.

Le module `ansible.builtin.service` permet de gérer l’état d’un service, par exemple `started`, `reloaded` ou `enabled`.

---

### 5.3 Fichier `handlers/main.yml`

Chemin :

```text
/home/toto/ansible/roles/apache/handlers/main.yml
```

Contenu :

```yaml
---
- name: Reload apache
  ansible.builtin.service:
    name: "{{ apache_service_name }}"
    state: reloaded
```

Le handler sera appelé uniquement si une tâche avec :

```yaml
notify: Reload apache
```

provoque un changement.

C’est adapté pour recharger Apache seulement quand sa configuration ou sa page HTML change.

---

### 5.4 Template Apache `apache-site.conf.j2`

Chemin :

```text
/home/toto/ansible/roles/apache/templates/apache-site.conf.j2
```

Contenu :

```apache
<VirtualHost *:{{ apache_listen_port }}>
    ServerName {{ apache_server_name }}
    DocumentRoot {{ apache_document_root }}

    <Directory {{ apache_document_root }}>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

---

### 5.5 Template HTML `index.html.j2`

Chemin :

```text
/home/toto/ansible/roles/apache/templates/index.html.j2
```

Contenu :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Apache déployé avec Ansible</title>
</head>
<body>
    <h1>Apache fonctionne sur {{ inventory_hostname }}</h1>
    <p>Déploiement réalisé avec Ansible et des modules ansible.builtin.</p>
    <p>Serveur cible : {{ ansible_host | default(inventory_hostname) }}</p>
</body>
</html>
```

---

## 6. Vérifications et lancement

Se placer dans le dossier du projet :

```bash
cd /home/toto/ansible
```

Vérifier l’inventaire :

```bash
ansible-inventory --list
```

Tester la connexion SSH Ansible :

```bash
ansible webservers -m ansible.builtin.ping
```

Résultat attendu :

```text
client-apache | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Vérifier la syntaxe du playbook :

```bash
ansible-playbook playbooks/apache.yml --syntax-check
```

Résultat attendu :

```text
playbook: playbooks/apache.yml
```

Lancer le déploiement :

```bash
ansible-playbook playbooks/apache.yml
```

Tester Apache depuis la machine de contrôle Ansible :

```bash
curl http://172.16.93.172
```

Résultat attendu dans la page HTML :

```html
<h1>Apache fonctionne sur client-apache</h1>
```

---

## 7. Commandes utiles de debug

Vérifier l’inventaire :

```bash
ansible-inventory --list
```

Tester la connexion :

```bash
ansible webservers -m ansible.builtin.ping
```

Vérifier la syntaxe :

```bash
ansible-playbook playbooks/apache.yml --syntax-check
```

Faire un dry-run :

```bash
ansible-playbook playbooks/apache.yml --check
```

Relancer le déploiement :

```bash
ansible-playbook playbooks/apache.yml
```

Tester l’accès HTTP :

```bash
curl http://172.16.93.172
```

---

## 8. Vérification côté client

Connexion au client :

```bash
ssh ansible@172.16.93.172
```

Vérifier le service Apache :

```bash
sudo systemctl status apache2
```

Tester la configuration Apache :

```bash
sudo apache2ctl configtest
```

Tester localement depuis le client :

```bash
curl http://localhost
```

---

## 9. Résultat attendu

À la fin du déploiement, le client doit avoir :

```text
Apache installé
Service apache2 démarré
Service apache2 activé au démarrage
Configuration Apache déployée
Page index.html déployée
Site accessible sur http://172.16.93.172
```
