# GitHub automatisé : recherche, documentation et construction du script

Ce document retrace notre démarche : les questions de départ, nos recherches, nos choix, la construction du script étape par étape et les difficultés rencontrées.

Le mode d'emploi du script est dans le [README](README.md).

---

## 1. Le sujet

Construire en Bash un outil qui automatise :

- la création du dépôt distant
- l'ajout des collaborateurs
- l'initialisation de l'arborescence locale (Job01, Job02…)
- le premier push

Compétences visées :

- Scripting Bash avancé (fonctions, gestion des erreurs, formulaires interactifs)
- Consommation d'une API REST authentifiée (API GitHub)
- Sécurisation d'un script shell (validation des entrées, gestion des secrets, moindre privilège)

---

## 2. Nos questions de départ

Avant de coder, on a listé ce qu'on ne savait pas :

- Le token doit-il être mis à chaque fois dans le script ?
- Gestion des secrets : quelle solution ? Auto-hébergée ou cloud ?
- Comment intégrer le token au script sans l'afficher ?
- Comment gérer les erreurs de connexion avec l'API ? Script en boucle ?
- Quelles informations demander à l'utilisateur ?
- Quels endpoints de l'API GitHub utiliser ?

---

## 3. Algorithme de départ

### Proposition d'Alex

Le script se lance et demande :

- le nom du projet
- le nombre de collaborateurs
- le nom des collaborateurs sur GitHub
- le nombre de jobs

Puis il se connecte à GitHub, crée le dépôt avec les collaborateurs et les jobs, et fait un `git init`, commit et push avec un README.

Point important dès le départ : **vérifier que chaque saisie est valide** (pas d'espace dans les noms, nombre de jobs = un nombre).

### Pseudo-code final

```
1. Vérifier les outils
2. Charger le token (fichier présent, droits 600, non vide)
3. Demander : login, dépôt, nb de jobs, collaborateurs
4. Valider les saisies
5. Créer le dépôt (API)
6. Ajouter les collaborateurs (API)
7. Créer JOB01…JOBn + README
8. git clone → add → commit → push
```

---

## 4. Recherches et choix

### 4.1 L'API REST GitHub

Une API, c'est un guichet : on envoie une demande (requête HTTP) avec sa pièce d'identité (le token), et le guichet répond par un code et des données (JSON).

| Action | Méthode + endpoint |
|---|---|
| Créer un dépôt | `POST /user/repos` |
| Ajouter un collaborateur | `PUT /repos/{owner}/{repo}/collaborators/{user}` |
| Vérifier le token | `GET /user` |

Codes de réponse utiles :

| Code | Sens |
|---|---|
| 201 | Créé (dépôt créé, invitation envoyée) |
| 204 | Déjà collaborateur |
| 401 | Token invalide ou expiré |
| 403 | Droits insuffisants |
| 422 | Données refusées (ex. dépôt déjà existant) |

### 4.2 Gestion des secrets : où mettre le token ?

**Réponse à la question d'Alex : non, le token ne doit jamais être écrit dans le script.**

Ce qu'on a appris :

- Un script est un fichier texte lisible. On ne peut pas « cacher » un token dedans (base64, découpage = de l'obfuscation, pas de la sécurité).
- Solution retenue (validée par notre formateur) : un **fichier hors du dépôt**, protégé en `600`, chargé dans une **variable d'environnement** au lancement.

Distinction importante qu'on a mis du temps à comprendre :

| Variable | Contient |
|---|---|
| `FICHIER_TOKEN` | le **chemin** du fichier (`~/.config/gitbash/env`) |
| `GITHUB_TOKEN` | le **token** lui-même, créé par le `source` du fichier |

Analogie : `FICHIER_TOKEN` = l'adresse du coffre, `GITHUB_TOKEN` = la clé qui est dedans.

Trajet du token :

```
fichier env (chmod 600)
   ↓ source
$GITHUB_TOKEN (en mémoire)
   ↓ curl -H "Authorization: Bearer ..."
API GitHub
```

Pourquoi le fichier contient `export GITHUB_TOKEN=...` et pas juste le token : `source` **exécute** le fichier comme des commandes Bash. Une ligne avec seulement le token serait lue comme une commande inconnue.

### 4.3 Les droits : 700 et 600

| Chiffre | Fichier | Dossier |
|---|---|---|
| 4 (r) | lire | voir la liste |
| 2 (w) | modifier | créer/supprimer dedans |
| 1 (x) | exécuter | **entrer** dans le dossier |

- Dossier en **700** : il faut le `x` pour entrer dedans.
- Fichier en **600** : le token n'est pas un programme, pas besoin de `x`.
- Les `00` : ni le groupe ni les autres n'ont accès.

### 4.4 Moindre privilège : le token

On a choisi un **token fine-grained** :

| Réglage | Choix | Pourquoi |
|---|---|---|
| Expiration | 30 jours | Limite les dégâts en cas de fuite |
| Repository access | All repositories | Le dépôt n'existe pas encore |
| Administration | Read and write | Créer le dépôt, inviter les collaborateurs |
| Contents | Read and write | Faire le premier push |

### 4.5 Environnement : pourquoi WSL

Premier essai dans PowerShell : erreur `mkdir : Impossible de trouver un paramètre positionnel acceptant l'argument «700»`.

On a compris que **PowerShell n'est pas Bash**. On est passés sur **WSL (Debian)**. Autre point : travailler dans `/home/...` et pas dans `/mnt/c/...` (le disque Windows), où `chmod` ne protège pas vraiment.

### 4.6 Authentifier le push

Le token sert à l'API (`curl`), mais `git push` a aussi besoin de s'identifier. Options étudiées :

| Méthode | Avantage | Inconvénient |
|---|---|---|
| Token dans l'URL | Simple | Écrit dans `.git/config` |
| `credential.helper store` | Ne redemande plus | Token en clair dans `~/.git-credentials` |
| Credential helper inline (`git -c`) | Rien n'est écrit | Syntaxe difficile à expliquer |
| Clé SSH | Accès séparés token / push | Préparation en plus |
| Saisie manuelle | Simple et sûr | Plus automatique |

On a retenu la méthode la plus simple à comprendre pour nous : le token dans l'adresse. On a identifié sa limite (voir partie 7).

### 4.7 Validation des entrées : les regex

Une regex est un **modèle** que le texte doit respecter. En Bash, on l'utilise avec `=~` dans `[[ ... ]]`.

| Symbole | Sens |
|---|---|
| `^` | début du texte |
| `$` | fin du texte |
| `[0-9]` | un chiffre |
| `{1,2}` | 1 ou 2 fois |

Exemple utilisé : `^[0-9]{1,2}$` pour le nombre de jobs.

Leçon apprise : **sans `^` et `$`**, `abc12` passe le test, car il *contient* des chiffres.

### 4.8 Git ignore les dossiers vides

Solutions trouvées :

- fichier `.gitkeep` (convention la plus utilisée)
- fichier `README.md` dans le dossier
- fichier `.gitignore` avec des règles

**Choix : un `README.md` dans chaque dossier JOB**, pour pouvoir y décrire le job.

---

## 5. Construction du script étape par étape

### Étape 1 : préparation et sécurité

```bash
FICHIER_TOKEN="$HOME/.config/gitbash/env"
set -euo pipefail
```

`set -euo pipefail` arrête le script à la première erreur, refuse les variables non définies et détecte les erreurs dans les pipes.

### Étape 2 : vérifier les outils

```bash
check_outil() {
    for outil in curl git; do
        if ! command -v "$outil" > /dev/null 2>&1; then
            echo "Erreur : $outil pas installe" >&2
            exit 1
        fi
    done
}
```

Une boucle passe chaque outil en revue. S'il manque, message clair et arrêt.

### Étape 3 : vérifier et charger le token

Trois contrôles :

1. Le fichier existe ? (`-f`)
2. Les droits sont `600` ? (`stat -c %a`)
3. Le token n'est pas vide ? (`-z "$GITHUB_TOKEN"`)

```bash
check_restriction() {
    if [ "$(stat -c %a "$FICHIER_TOKEN")" != "600" ]; then
        echo "Erreur : le fichier n'a pas les bonnes restrictions" >&2
        exit 1
    fi
}
```

Erreur corrigée : au début, on testait le fichier avec `cat`. Mais le fichier contient `export GITHUB_TOKEN=`, donc il n'est jamais vide même sans token. On teste maintenant **la variable** avec `-z`.

### Étape 4 : formulaire et validation

- Boucles `while` : on redemande tant que la saisie est vide ou contient un espace.
- Nombre de jobs : regex `^[0-9]{1,2}$`, redemandé tant qu'il est invalide.

Erreur corrigée : notre première condition utilisait `&&` (« contient un espace **et** est vide ») : c'est impossible, donc la question n'était jamais posée. Il fallait `||`.

### Étape 5 : créer le dépôt distant

```bash
curl -i \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -d "{\"name\": \"$repoName\", \"auto_init\": true, \"private\": true}" \
  https://api.github.com/user/repos
```

Erreurs corrigées :

- Avec des guillemets simples `'...'`, `$repoName` n'était pas remplacé : GitHub recevait littéralement « $repoName ».
- Des espaces après les `\` coupaient la commande.

### Étape 6 : ajouter les collaborateurs

`PUT /repos/{owner}/{repo}/collaborators/{user}` avec la permission `push` (les collaborateurs peuvent pousser et tirer). Le collaborateur reçoit une **invitation** à accepter.

### Étape 7 : arborescence

Une boucle `while` crée les dossiers `JOBxx`, avec un zéro devant les numéros inférieurs à 10, et un `README.md` dans chacun.

Erreur corrigée : un `exit 0` à la fin de la fonction arrêtait **tout le script**. Remplacé par `return`.

### Étape 8 : Git

```
git clone → createDirectory → git add → git commit → git push
```

Comme le dépôt est créé avec `auto_init: true`, il contient déjà un commit : on le clone, puis on ajoute les dossiers.

### Étape 9 : `main`

`main` appelle chaque fonction dans l'ordre. Analogie : une chaîne de montage, chaque fonction est un poste.

---

## 6. Difficultés rencontrées

| Difficulté | Ce qu'on a appris |
|---|---|
| Commandes Bash dans PowerShell | Deux langages différents → WSL |
| Travail dans `/mnt/c/` | `chmod` peu fiable sur le disque Windows |
| Token vide (fichier de 23 octets) | Le collage n'était pas passé ; vérifier la taille du fichier |
| `$FICHIER_TOKEN` utilisé comme token | Chemin ≠ token |
| `${$VARIABLE}` | Pas de `$` à l'intérieur des accolades |
| Guillemets simples dans le JSON | Pas de remplacement des variables |
| `\` suivi d'un espace | La commande est coupée |
| `exit 0` dans une fonction | Arrête tout le script |
| Condition `while` avec `&&` | Condition impossible → `\|\|` |
| Regex sans `^` et `$` | Valide des saisies incorrectes |
| Dossiers vides | Ignorés par Git → README.md |
| Fonctions jamais appelées | Une fonction ne fait rien sans appel |

---

## 7. Limites connues et améliorations possibles

| Limite | Amélioration possible |
|---|---|
| Le token est enregistré dans `.git/config` après le clone | Clé SSH, ou remettre une adresse sans token après le clone |
| Pas de vérification du code HTTP après `curl` | Récupérer le code avec `-w "%{http_code}"` |
| Formulaire avec `read` | Passer à `dialog` |
| Validation limitée aux espaces | Regex complète sur les noms |
| Pas de vérification du token avant de commencer | Appel `GET /user` |
| Pas de nettoyage en cas d'erreur | `trap` |

---

## 8. Sources

### Sécurité et secrets

- Gérer les fichiers .env sur Ubuntu : https://oneuptime.com/blog/post/2026-03-02-how-to-manage-env-files-securely-on-ubuntu/view
- Lire un fichier .env en Bash : https://askubuntu.com/questions/1389904/read-from-env-file-and-set-as-bash-variables
- Droits et chmod : https://linuxize.com/post/chmod-command-in-linux/
- Credential helpers Git (doc officielle) : https://git-scm.com/docs/gitcredentials

### API GitHub

- Ajouter un collaborateur (doc officielle) : https://docs.github.com/fr/rest/collaborators/collaborators?apiVersion=2026-03-10#add-a-repository-collaborator
- Ajouter des collaborateurs : https://www.blinkops.com/blog/how-to-add-collaborators-to-your-github-repository
- Créer un dépôt avec GitHub CLI : https://cli.github.com/manual/gh_repo_create

### Bash

- Scripts shell : https://blog.stephane-robert.info/docs/admin-serveurs/linux/exploiter/scripts/shell/
- Regex en Bash : https://fr.wikibooks.org/wiki/Programmation_Bash/Regex
- Vérifier un fichier ou un dossier : https://www.it-connect.fr/verifier-la-presence-dun-repertoire-ou-dun-fichier/
- Tester si un fichier est vide : https://linuxsimply.com/bash-scripting-tutorial/conditional-statements/if/check-if-file-is-empty/

### Scripts d'inspiration

- Script d'initialisation de dépôt : https://dev.to/cosmicwanderer7/github-repository-initialization-script-12pp
- Créer un dépôt distant depuis le terminal : https://medium.com/thelearningcurve/set-up-a-remote-github-repository-from-terminal-802ad62fb550
- Modèles .gitignore : https://github.com/github/gitignore
