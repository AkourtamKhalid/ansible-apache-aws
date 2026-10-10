# Déploiement automatisé d'Apache sur AWS EC2 avec Ansible (Ubuntu + RedHat)

Projet réalisé chez **AUBERTECH-Chapiock**.

**Objectif :** depuis une seule machine de contrôle, déployer et configurer automatiquement un serveur web Apache sur **deux distributions Linux différentes** hébergées sur **AWS EC2**, avec **un seul rôle Ansible**, une page HTML personnalisée par serveur, et un playbook **idempotent**.

> Mots-clés : Ansible · Infrastructure as Code · AWS EC2 · VPC · Subnet public · Route Table · Internet Gateway · Security Group · SSH · Linux (Ubuntu / Amazon Linux) · Apache · Jinja2 · Rôles · Facts · Idempotence · Terraform

---

## 1. Architecture

```
                    Machine de contrôle (WSL Ubuntu + Ansible)
                                    │
                         SSH (clé .pem, port 22)
                                    │
        ┌───────────────────────────┴───────────────────────────┐
        │                AWS eu-west-3 (Paris)                   │
        │   VPC 50.20.0.0/16                                     │
        │   └── Subnet public 50.20.18.0/24                      │
        │        Route 0.0.0.0/0 → Internet Gateway              │
        │        Security Group : 22 + 80 depuis mon IP /32      │
        │                                                        │
        │        ┌──────────────────┐   ┌──────────────────┐    │
        │        │ web-ubuntu        │   │ web-centos        │    │
        │        │ Ubuntu 24.04      │   │ Amazon Linux 2023 │    │
        │        │ famille Debian    │   │ famille RedHat    │    │
        │        │ apt + apache2     │   │ dnf + httpd       │    │
        │        └──────────────────┘   └──────────────────┘    │
        └────────────────────────────────────────────────────────┘
```

- **Machine de contrôle :** Ansible installé uniquement sur WSL. Ansible est **agentless** : les serveurs n'ont besoin que de SSH et de Python.
- **Réseau AWS :** subnet rendu public par une table de routage `0.0.0.0/0 → IGW`, accès SSH et HTTP restreints à mon IP via le Security Group.
- **Deux familles d'OS :** le cœur du projet est de gérer leurs différences (gestionnaire de paquets, nom du paquet et du service, démarrage automatique ou non) avec un seul rôle.

> Choix technique : CentOS 8 étant en fin de vie, j'ai utilisé **Amazon Linux 2023**, de la même famille RedHat (`dnf`, `httpd`). Le rôle se base sur le fact `ansible_os_family`, il fonctionne donc aussi sur CentOS, Rocky ou AlmaLinux.

---

## 2. Structure du projet

```
ansible/
├── ansible.cfg                  # config locale : pointe vers l'inventaire
├── inventory.ini                # groupe apache_servers + paramètres de connexion
├── site.yml                     # playbook principal (point d'entrée)
└── roles/
    └── apache/
        ├── defaults/main.yml    # variables : paquets/services par famille, server_name
        ├── tasks/
        │   ├── main.yml         # orchestration : choix par os_family + template
        │   ├── ubuntu.yml       # famille Debian : apt + service
        │   └── centos.yml       # famille RedHat : dnf + service
        ├── templates/
        │   └── index.html.j2    # page dynamique Jinja2
        ├── files/
        │   └── index.html       # version statique (étape copy)
        └── handlers/main.yml    # prévu pour un redémarrage sur changement de config
```

---

## 3. Comment ça s'enchaîne

1. `ansible-playbook site.yml` lit **`ansible.cfg`**, qui pointe vers **`inventory.ini`**.
2. L'inventaire définit le groupe **`apache_servers`** et, pour chaque hôte, `ansible_host`, `ansible_user`, `ansible_python_interpreter`, ainsi que la clé SSH partagée via `[apache_servers:vars]`.
3. **`site.yml`** cible ce groupe, active `become` (sudo) et applique le rôle **`apache`**.
4. **Gathering Facts :** Ansible collecte les facts de chaque serveur (`os_family`, `distribution`…).
5. Le rôle charge **`defaults/main.yml`**, puis exécute **`tasks/main.yml`** :
   - `include_tasks: ubuntu.yml` si `os_family == "Debian"` → `apt` + `apache2` ;
   - `include_tasks: centos.yml` si `os_family == "RedHat"` → `dnf` + `httpd` ;
   - module **`template`** → génère `/var/www/html/index.html` pour chaque serveur.
6. La page affiche **« Bienvenue sur web-ubuntu »** ou **« Bienvenue sur web-centos »**. Le nom vient de `server_name: "{{ inventory_hostname }}"` (magic variable), la distribution vient des facts. Aucun nom n'est écrit en dur dans le HTML.

---

## 4. Notions Ansible mises en œuvre

| Notion | Utilisation dans le projet |
|---|---|
| **Inventaire et groupes** | `[apache_servers]`, variables de groupe `:vars` |
| **Configuration locale** | `ansible.cfg` dans le projet, prioritaire sur `/etc/ansible/ansible.cfg` |
| **Commandes ad-hoc** | `ping`, `setup`, `command`, `uri` pour tester et vérifier |
| **Facts** | `ansible_os_family` pour choisir apt ou dnf, `ansible_distribution` affichée sur la page |
| **Magic variables** | `inventory_hostname` → `server_name` |
| **Rôle** | structure standard `defaults/`, `tasks/`, `templates/`, `files/`, `handlers/` |
| **Conditions** | `when` + `include_tasks` selon la famille d'OS |
| **Modules** | `apt`, `dnf`, `service`, `copy`, `template`, `uri`, `command` |
| **Privilèges** | `become: true` |
| **Templates Jinja2** | page HTML personnalisée par serveur |
| **Idempotence** | état déclaratif (`state: present`, `state: started`), `cache_valid_time` |
| **Simulation** | `--check --diff` (équivalent d'un `terraform plan`) |
| **Sécurité** | authentification par clé SSH, aucun secret dans le dépôt, Security Group restreint à une IP |

---

## 5. Résultats

**1er passage :** installation sur les deux familles, `failed=0`. On observe la différence entre les OS : le démarrage d'Apache est `ok` sur Ubuntu (démarré automatiquement après l'installation) et `changed` sur la famille RedHat (Ansible a dû le démarrer).

**Vérification depuis chaque serveur :**
```bash
ansible web-ubuntu -m ansible.builtin.command -a "systemctl is-active apache2"   # active
ansible web-centos -m ansible.builtin.command -a "systemctl is-active httpd"     # active
ansible apache_servers -m ansible.builtin.uri -a "url=http://localhost return_content=true"   # 200
```

**Idempotence (2e passage, sans modification) :**
```
web-centos : ok=5  changed=0  unreachable=0  failed=0  skipped=1
web-ubuntu : ok=5  changed=0  unreachable=0  failed=0  skipped=1
```

**Résilience :** après un redémarrage des instances EC2, Apache repart seul (`enabled: true`) et un nouveau passage donne toujours `changed=0`. Seules les IP publiques sont à mettre à jour, à un seul endroit : l'inventaire.

---

## 6. Problèmes rencontrés et diagnostic

| Symptôme | Cause | Résolution |
|---|---|---|
| SSH en timeout alors que le Security Group et l'IP étaient corrects | La table de routage avec `0.0.0.0/0 → IGW` n'était pas **associée** au subnet, qui utilisait la table principale du VPC | Association explicite de la table de routage au subnet |
| `UNREACHABLE` le lendemain | IP publiques des EC2 changées après redémarrage (et IP du poste changée) | Mise à jour de `inventory.ini`, du Security Group et de `known_hosts` (`ssh-keyscan`) |
| Warning sur l'interpréteur Python découvert | Ansible devinait le Python distant | `ansible_python_interpreter=/usr/bin/python3` dans l'inventaire |
| `FAILED : Could not find 'index.html.j2'` | Faute de frappe dans le nom du template | Renommage ; lecture des chemins de recherche affichés par Ansible |
| HTTP 403 sur la famille RedHat | `/var/www/html` vide après installation | Déploiement de la page d'accueil (copy, puis template) |

`UNREACHABLE` signale un problème réseau (pas de connexion SSH), alors que `FAILED` signale une tâche en erreur sur un hôte joignable.

---

## 7. Passage à l'échelle : forks, serial, job slicing

Le projet cible 2 serveurs, mais le même rôle s'applique à un parc plus large. Voici les leviers que j'ai étudiés :

- **`forks`** (5 par défaut) : nombre d'hôtes traités **en parallèle** par la machine de contrôle, pour chaque tâche. Avec 100 serveurs et `forks = 50`, une tâche s'exécute en 2 vagues au lieu de 20. La limite est la capacité de la machine de contrôle (CPU, RAM, connexions SSH). Réglage via `ansible.cfg`, `-f` ou `ANSIBLE_FORKS`.
- **`serial`** : à l'inverse, limite volontairement le nombre d'hôtes traités à la fois, pour des mises à jour progressives (*rolling updates*) sans interruption de service.
- **Job slicing** (AWX / Ansible Automation Platform) : découpe un job en N jobs, chacun sur une partie de l'inventaire, répartis sur plusieurs nœuds d'exécution. Le parallélisme total vaut slices × forks. Le rôle s'y prête bien : chaque hôte se configure indépendamment, sans dépendre de `hostvars` d'autres hôtes.

---

## 8. Lien avec Terraform

Dans un projet précédent de la même formation, j'ai décrit en **Terraform** une architecture AWS comparable : VPC existant lu via `data`, subnets public et privé, NAT Gateway, Security Groups, EC2 frontend (Angular/Nginx) et backend (Flask), avec `user_data` et `templatefile()`. Je l'ai ensuite découpée en **modules** (`subnets`, `gateways`, `instance`) en migrant le state avec `terraform state mv`, puis j'ai déplacé le **state dans S3** (versioning, chiffrement AES256, blocage de l'accès public, `use_lockfile`).

Ici, faute de temps, les deux EC2 ont été créées via la console AWS. Les deux outils sont complémentaires :

| | Terraform | Ansible |
|---|---|---|
| Rôle | **Provisionner** l'infrastructure (réseau, EC2) | **Configurer** les machines (paquets, services, fichiers) |
| État | State file (local ou S3) | Pas de state : compare l'état voulu à l'état réel à chaque passage |
| Prévisualisation | `terraform plan` | `ansible-playbook --check --diff` |
| Templates | `templatefile()` | module `template` (Jinja2) |

---

## 9. Lancer le projet

Prérequis : Ansible sur la machine de contrôle (Linux / WSL), deux EC2 joignables en SSH, ports 22 et 80 ouverts depuis votre IP.

```bash
# 1. Renseigner les IP et utilisateurs dans inventory.ini, ainsi que le chemin de votre clé SSH

# 2. Vérifier
ansible --version                                  # config file = …/ansible.cfg
ansible-inventory --graph                          # @apache_servers → 2 hôtes
ansible apache_servers -m ansible.builtin.ping     # SUCCESS

# 3. Déployer
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff
ansible-playbook site.yml

# 4. Tester
curl http://<IP_PUBLIQUE>
```

> Les IP de `inventory.ini` sont des IP publiques éphémères qui n'existent plus. La clé privée SSH n'est **pas** versionnée : seul son chemin apparaît dans l'inventaire.

---

## 10. Améliorations envisagées

- Provisionner les EC2 avec **Terraform** et générer l'inventaire automatiquement (plugin d'inventaire dynamique `amazon.aws.aws_ec2`, basé sur les tags).
- Utiliser des **Elastic IP** ou l'inventaire dynamique pour ne plus dépendre d'IP éphémères.
- Ajouter un **handler** `reload apache` pour les changements de configuration Apache (virtual hosts).
- **Ansible Vault** dès qu'un secret entre dans le projet.
- Une **CI GitLab / GitHub Actions** avec `ansible-lint` et `--syntax-check` à chaque push.
- Des tests de rôle avec **Molecule**.
