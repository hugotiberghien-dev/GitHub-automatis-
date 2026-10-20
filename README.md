# GitHub automatisé

Automatiser, en Bash, la création d'un dépôt GitHub et de ses jobs, sécurisé dès la conception.

## Prérequis

- WSL
- curl
- git

### WSL

**Pourquoi :** le script est en Bash. Windows ne parle pas Bash. `chmod 600` ne protège vraiment le token que sur un système de fichiers Linux.

### curl

**Pourquoi :** c'est lui qui parle à l'API GitHub :

- créer le dépôt (`POST /user/repos`)
- inviter les collaborateurs (`PUT .../collaborators`)

### git

**Pourquoi :**

- `git clone`, `add`, `commit`, `push`

### Installation (dans WSL)

```bash
sudo apt update && sudo apt install -y curl git
```

## Créer le token GitHub

Le script utilise un **token fine-grained** pour se connecter à l'API GitHub.

### Étapes

1. Sur GitHub : photo de profil → **Settings**
2. **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
3. Cliquer sur **Generate new token**
4. Remplir les réglages ci-dessous
5. Cliquer sur **Generate token** et **copier le token** (il ne sera plus affiché)

### Réglages

| Réglage | Choix | Pourquoi |
|---|---|---|
| Expiration | Courte (30 jours) | Limite les dégâts en cas de fuite |
| Repository access | All repositories | Le dépôt n'existe pas encore au moment de la création |
| Administration | Read and write | Créer le dépôt et inviter les collaborateurs |
| Contents | Read and write | Faire le premier push |
| Metadata | Read (automatique) | Obligatoire pour toutes les requêtes |

### Pourquoi ces choix

On applique le **moindre privilège** : le token a seulement les droits dont le script a besoin, et pour une durée limitée.

Si le token fuite : le **révoquer immédiatement** sur GitHub, puis en créer un nouveau.

## Stocker le token

Le token est rangé dans un fichier, **hors du dépôt**, lisible uniquement par nous.

Dans le terminal WSL :

```bash
mkdir -p ~/.config/gitbash
chmod 700 ~/.config/gitbash
nano ~/.config/gitbash/env
chmod 600 ~/.config/gitbash/env
```

| Commande | Rôle |
|---|---|
| `mkdir -p ~/.config/gitbash` | Crée le dossier qui contiendra le fichier du token |
| `chmod 700 ~/.config/gitbash` | Seul notre utilisateur peut entrer dans le dossier |
| `nano ~/.config/gitbash/env` | Ouvre le fichier. Y écrire : `export GITHUB_TOKEN=votre_token` |
| `chmod 600 ~/.config/gitbash/env` | Seul notre utilisateur peut lire et modifier le fichier |

Dans nano : coller avec `Ctrl+Shift+V`, enregistrer avec `Ctrl+O` puis `Entrée`, quitter avec `Ctrl+X`.

## Lancer le script

```bash
chmod +x gitbash.sh
./gitbash.sh
```

Le dépôt est cloné dans le dossier où le script est lancé.

Le script demande :

| Question | Règle |
|---|---|
| Nom utilisateur GitHub | Sans espace, non vide |
| Nom du projet | Sans espace, non vide |
| Nombre de jobs | 1 ou 2 chiffres (redemande tant que c'est invalide) |
| Collaborateurs | Nom GitHub du collaborateur |

## Ce que fait le script

1. Vérifie que `curl` et `git` sont installés
2. Vérifie que le fichier du token existe
3. Vérifie que le fichier est en `600`
4. Charge le token et vérifie qu'il n'est pas vide
5. Pose les questions du formulaire et valide les réponses
6. Crée le dépôt privé sur GitHub (`POST /user/repos`)
7. Invite les collaborateurs (`PUT /repos/{owner}/{repo}/collaborators/{user}`)
8. Clone le dépôt, crée les dossiers `JOBxx` avec un `README.md`, puis commit et push

## Sécurité

### Gestion des secrets

- Le token n'est **jamais écrit dans le script**
- Il est stocké dans `~/.config/gitbash/env`, hors du dépôt
- Le script refuse de continuer si le fichier n'est pas en `600`
- Le script s'arrête si le token est vide
- `set -euo pipefail` arrête le script à la première erreur

### Moindre privilège

- Token **fine-grained** avec une expiration courte
- Seulement les permissions nécessaires : Administration et Contents
- Les collaborateurs sont invités avec la permission `push`

### Validation des entrées

- Nom utilisateur et nom du projet : pas d'espace, pas de valeur vide
- Nombre de jobs : regex `^[0-9]{1,2}$`, le script redemande tant que la saisie est invalide

### Point d'attention

Le clone utilise une adresse contenant le token. Git l'enregistre dans `.git/config` du dépôt local. Ce fichier n'est pas envoyé sur GitHub, mais il reste lisible sur la machine.
