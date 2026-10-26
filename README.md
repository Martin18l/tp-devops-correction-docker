# TP DevOps Correction Docker

Correction de la partie Docker du module DevOps. Amusez-vous bien avec GitHub Actions !
j'ai modifié

## Déploiement avec Ansible

Le dossier `ansible/` installe Docker sur Ubuntu 24.04 et déploie la base PostgreSQL,
l'API et le proxy HTTP dans le réseau Docker `my-network`. Les données PostgreSQL
sont conservées dans le volume `db-volume`. Le proxy expose le port 80 et route
les requêtes vers l'API.

### Préparer l'accès au serveur

Le serveur doit être sous Ubuntu 24.04, accessible en SSH avec un utilisateur
disposant des droits `sudo`. La clé privée SSH et le mot de passe PostgreSQL ne
doivent pas être ajoutés au dépôt ni envoyés dans un message.

Ansible doit être lancé depuis Linux, macOS ou WSL (le serveur distant est
Ubuntu). Pour un lancement manuel, définir les variables d'environnement
suivantes dans le shell depuis lequel Ansible est exécuté :

| Variable | Contenu |
| --- | --- |
| `DEPLOY_HOST` | Nom DNS ou adresse IP du serveur |
| `DEPLOY_USER` | Utilisateur SSH (par exemple `ubuntu`) |
| `ANSIBLE_PRIVATE_KEY_FILE` | Chemin local vers la clé SSH privée |
| `DOCKERHUB_USERNAME` | Compte Docker Hub propriétaire des images |
| `POSTGRES_PASSWORD` | Mot de passe PostgreSQL |
| `IMAGE_TAG` | Tag immuable des images, généralement le SHA du commit |

Créer un environnement virtuel, installer Ansible et la collection Docker, puis
vérifier l'accès et le playbook :

```sh
cd ansible
python3 -m venv .venv
. .venv/bin/activate
python -m pip install ansible
ansible-galaxy collection install -r requirements.yml
ansible all -m ansible.builtin.ping
ansible all -m ansible.builtin.setup -a "filter=ansible_distribution*"
ansible-playbook --syntax-check playbook.yml
ansible-playbook playbook.yml
```

L'inventaire `inventories/setup.yml` lit sa cible et ses variables sensibles dans
l'environnement. L'interpréteur Python distant est fixé à `/usr/bin/python3`,
où le rôle Docker installe le SDK Python requis par `community.docker`.

### Déploiement continu GitHub Actions

Le job `deploy` s'exécute après la publication des images, uniquement sur un
push vers `main`. Les images sont publiées avec le SHA Git et le déploiement
utilise le SHA exact du workflow, plutôt qu'un tag mutable tel que `latest`.
Les builds des pull requests ne publient pas d'images et ne lancent pas de
déploiement.

Créer l'environnement GitHub Actions `production`, puis y configurer les secrets
suivants (le workflow utilise également le secret de dépôt existant
`DOCKERHUB_USERNAME`; le job de publication a aussi besoin du secret existant
`DOCKERHUB_TOKEN`) :

| Secret | Contenu |
| --- | --- |
| `DEPLOY_HOST` | Nom DNS ou adresse IP du serveur |
| `DEPLOY_USER` | Utilisateur SSH avec accès `sudo` |
| `DEPLOY_SSH_PRIVATE_KEY` | Clé privée dédiée au déploiement |
| `DEPLOY_KNOWN_HOSTS` | Clé d'hôte SSH vérifiée, au format `known_hosts` |
| `POSTGRES_PASSWORD` | Mot de passe PostgreSQL |

Restreindre l'environnement `production` à la branche `main` et configurer des
approbateurs obligatoires si le dépôt GitHub le permet. Pour réduire le risque
de déploiement d'une image compromise, limiter l'accès aux secrets, utiliser un
token Docker Hub à privilèges minimaux, protéger les changements vers `main`,
et ajouter l'analyse et la signature des images. Le SHA rend le déploiement
reproductible, mais ne garantit pas à lui seul l'innocuité de l'image.

Si PostgreSQL a déjà été initialisé dans `db-volume`, changer
`POSTGRES_PASSWORD` ne modifie pas le mot de passe existant dans la base. Il
faut conserver le mot de passe initial ou le faire tourner séparément dans
PostgreSQL.