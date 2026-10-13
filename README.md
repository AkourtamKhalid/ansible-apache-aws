# Ansible : déploiement Apache multi-OS avec pare-feu et VirtualHosts

Rôle Ansible qui déploie et sécurise un serveur web **Apache** sur deux familles Linux différentes,
**Ubuntu** (Debian) et **Amazon Linux / CentOS** (RedHat), depuis une seule machine de contrôle.
Il automatise les tutoriels DigitalOcean d'installation d'Apache sur Ubuntu et sur CentOS.

## Versions

- **v1.0** : installation et démarrage d'Apache sur les deux OS, page d'accueil personnalisée (template Jinja2), idempotence.
- **v2.0** : ajout du pare-feu (UFW / firewalld), des VirtualHosts et des handlers.

## Ce que fait le rôle

| | Ubuntu | Famille RedHat |
|---|---|---|
| Installation | `apt` : `apache2` | `dnf` : `httpd` |
| Service | démarré + activé au boot | démarré + activé au boot |
| Pare-feu | UFW : OpenSSH + Apache (80) | firewalld : ssh + http |
| Site | `/var/www/<serveur>` | `/var/www/<serveur>/html` + `/log` |
| VirtualHost | `sites-available` + `a2ensite` / `a2dissite` | `sites-available` / `sites-enabled` + `IncludeOptional` + lien symbolique |
| Page d'accueil | template Jinja2 personnalisé par serveur | idem |
| Redémarrage | handler, seulement si la config change | handler |

Le choix entre les deux branches se fait automatiquement avec le fact `ansible_os_family`.

## Structure

```
.
├── ansible.cfg
├── inventory.ini.example
├── site.yml
└── roles/apache/
    ├── defaults/main.yml        # noms de paquets/services, server_name
    ├── handlers/main.yml        # Redémarrer apache2 / httpd
    ├── tasks/
    │   ├── main.yml             # choix selon os_family
    │   ├── ubuntu.yml           # apt, UFW, VirtualHost Ubuntu
    │   └── centos.yml           # dnf, firewalld, VirtualHost RedHat
    └── templates/
        ├── index.html.j2
        ├── vhost-ubuntu.conf.j2
        └── vhost-centos.conf.j2
```

## Utilisation

```bash
cp inventory.ini.example inventory.ini     # renseigner les IP publiques
ansible apache_servers -m ansible.builtin.ping
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff   # simulation
ansible-playbook site.yml
```

Prérequis : collections `community.general` (module ufw) et `ansible.posix` (module firewalld).

## Résultats

- `http://<ip-ubuntu>` : « Bienvenue sur web-ubuntu »
- `http://<ip-centos>` : « Bienvenue sur web-centos »
- 2e exécution : `changed=0` et aucun handler déclenché, le rôle est idempotent.

## Notions mises en œuvre

Inventaire et groupes, facts (`os_family`), rôles, variables `defaults`, `when`, `include_tasks`,
templates Jinja2, magic variables (`inventory_hostname`), handlers et `notify`, `loop`, `lineinfile`,
module `file` (dossiers, liens), `creates` / `removes`, `changed_when`, collections, idempotence, check mode.

## Sécurité

- Authentification par clé SSH : aucun mot de passe dans le code.
- Défense en profondeur : Security Group AWS + pare-feu système (UFW / firewalld).
- Moindre privilège : seuls les ports 22 et 80 sont ouverts ; pages en `root:root`, lisibles par Apache.

## Environnement

AWS EC2 (eu-west-3) : Ubuntu 24.04 et Amazon Linux 2023 (famille RedHat, CentOS 8 étant en fin de vie).
Machine de contrôle : WSL Ubuntu, ansible-core 2.20.
