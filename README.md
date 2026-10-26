# TP DevOps Correction Docker

Correction de la partie Docker du module DevOps. Amusez-vous bien avec GitHub Actions !
j'ai modifié

## Déploiement avec Ansible

Le dossier `ansible/` installe Docker sur le serveur Debian du TD et déploie la base PostgreSQL,
l'API et le proxy HTTP dans le réseau Docker `my-network`. Les données PostgreSQL
sont conservées dans le volume `db-volume`. Le proxy expose le port 80 et route
les requêtes vers l'API.

### Préparer l'accès au serveur

Ansible s'exécute dans Ubuntu sous WSL sur ton PC et se connecte en SSH au
serveur Debian distant fourni pour le TD. Utilise l'utilisateur `admin` et le
domaine qui t'a été attribué. La clé privée SSH et le mot de passe PostgreSQL
ne doivent pas être ajoutés au dépôt ni envoyés dans un message. Le rôle Docker
retire aussi Apache s'il est installé, car le proxy du projet utilise le port 80.

Pour tester d'abord la connexion avec l'inventaire du TD :

```sh
ansible all -i /etc/ansible/hosts -m ping --private-key="$HOME/chemin/vers/ta-cle" -u admin
```

Après le `pong`, vérifier les informations du serveur :

```sh
ansible all -i /etc/ansible/hosts -m ansible.builtin.setup -a "filter=ansible_distribution*"
```

Ouvre ensuite `http://<ton-domaine-takima>` dans le navigateur Windows. Cette
étape vérifie Apache installé dans le TD; le déploiement de l'application le
remplacera par le proxy du projet.

Pour utiliser le playbook du projet, définir les variables d'environnement
suivantes dans le shell WSL depuis lequel Ansible est exécuté :

| Variable | Contenu |
| --- | --- |
| `DEPLOY_HOST` | Nom DNS ou adresse IP du serveur |
| `DEPLOY_USER` | Utilisateur SSH (`admin` pour le serveur du TD) |
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
| `DEPLOY_HOST` | Domaine ou adresse IP du serveur |
| `DEPLOY_USER` | Utilisateur SSH (`admin`) avec accès `sudo` |
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