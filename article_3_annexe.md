# Annexe : Initialiser le dépôt Git et le connecter à GitHub

<br>

## Pourquoi cette annexe

Dans l'article 3, on a introduit le tag `sha-$GIT_SHA`, dérivé du hash de commit courant. Ce mécanisme suppose un prérequis qu'on n'a jamais posé explicitement : que le projet vit déjà dans un dépôt Git, avec un historique de commits et un remote GitHub configuré. Si ce n'est pas encore le cas, cette annexe comble le trou avant de continuer. Et c'est toujours bon de réviser les basiques !

<br>

## Étape 1 : Initialiser le dépôt local

Depuis la racine du projet :

```bash
git init
```

Cette commande crée un sous-dossier caché `.git/` qui contiendra tout l'historique du projet. Rien n'est encore suivi ni committé à ce stade, c'est l'équivalent d'un `mkdir` pour un dépôt : la structure existe, mais elle est vide.

### Configurer l'identité Git (si ce n'est pas déjà fait globalement)

Chaque commit est signé avec un nom et un e-mail. Si git n'a jamais été configuré sur cette machine :

```bash
git config --global user.name "Nom ou pseudo"
git config --global user.email "email@exemple.com"
```

`--global` applique ce réglage à tous tes dépôts sur la machine, pas seulement celui-ci. Pour vérifier la configuration active à tout moment :

```bash
git config --list
```

<br>

## Étape 2 : Vérifier le `.gitignore`

Avant le premier commit, il faut s'assurer qu'aucun fichier sensible ou inutile ne sera suivi par Git. C'est le pendant du `.dockerignore` vu à l'article 2, mais pour Git plutôt que pour le contexte de build Docker. Les deux fichiers se recoupent largement, mais servent des mécanismes différents et doivent tous les deux exister.

```
# Dépendances
node_modules/

# Build Next.js
.next/
out/

# Variables d'environnement : ne jamais committer de secrets
.env.local
.env*.local

# Éditeurs / IDE
.vscode/
.idea/

# Système
.DS_Store
Thumbs.db
```

**Point d'attention particulier** : `.env.local` (qui contient la variable `NEXT_PUBLIC_MAPBOX_TOKEN` en local) ne doit **jamais** être committé, même si on a établi que ce token n'est pas un secret critique une fois injectée dans le bundle JS. La discipline de ne jamais committer de fichier `.env*` doit rester systématique, sans exception au cas par cas.

Si le fichier `.gitignore` n'existe pas encore, mieux vaut le créer avant le premier `git add`. Un fichier une fois committé reste dans l'historique même si on l'ajoute au `.gitignore` après coup, et il faudrait alors une opération de nettoyage a posteriori, plus délicate.

<br>

## Étape 3 : Premier commit

```bash
# Vérifier ce qui sera suivi (contrôle avant l'ajout)
git status

# Ajouter l'ensemble des fichiers non ignorés
git add .

# Vérifier une dernière fois ce qui va être committé
git status

# Premier commit
git commit -m "Initial commit"
```

Le `git status` intermédiaire n'est pas superflu : c'est le moment de repérer un fichier qui n'aurait pas dû être suivi (un `.env.local` mal ignoré, par exemple) avant qu'il n'entre dans l'historique.

<br>

## Étape 4 : Créer le dépôt distant sur GitHub


### Depuis l'interface web GitHub

1. Sur [github.com](https://github.com), cliquer sur **New repository**
2. Nommer le dépôt, ici **WestBaliDrivers**
3. Laisser le dépôt **vide**. Ne pas cocher "Add a README", "Add .gitignore", ou "Add license". Ces options créeraient un commit initial côté GitHub, qui entrerait en conflit avec l'historique local déjà existant.
4. Choisir **Private**. Même si ce n'est "qu'un" site vitrine, il reste construit pour le compte d'un réseau de chauffeurs réel. Rendre son code public reviendrait à donner gratuitement le "patrimoine numérique" d'une petite activité commerciale. Il est donc à mon sens de bon ton de conserver cela privé.


<br>

## Étape 5 : Configurer le remote 

GitHub affiche, juste après la création du dépôt vide, les commandes à exécuter. Elles ressemblent à ceci :

```bash
git remote add origin https://github.com/lolecat/WestBaliDrivers.git
```

`origin` est simplement un alias conventionnel pour désigner ce dépôt distant. Rien n'oblige à ce nom précis, mais c'est la convention quasi universelle, à conserver sauf raison particulière.

Vérification :

```bash
git remote -v
```

```
origin  https://github.com/lolecat/WestBaliDrivers.git (fetch)
origin  https://github.com/lolecat/WestBaliDrivers.git (push)
```

<br>

## Étape 6 : Authentification GitHub

Depuis 2021, GitHub n'accepte plus les mots de passe classiques pour l'authentification Git en HTTPS. Deux options sont possibles :

### HTTPS + Personal Access Token (PAT)

Au premier `git push`, Git demandera un nom d'utilisateur puis un mot de passe. C'est à cet endroit qu'un **token** (généré depuis GitHub → Settings → Developer settings → Personal access tokens) doit être collé à la place d'un mot de passe.

### SSH

Si l'on possède déjà une paire de clef, s'assurer de l'avoir dans le `~/.ssh/` de sa machine hôte.  
Sinon, on  génère une paire de clés :

```bash
ssh-keygen -t ed25519 -C "email@exemple.com"
```

La clé publique (`~/.ssh/id_ed25519.pub`) est ensuite à coller dans GitHub → Settings → SSH and GPG keys → New SSH key.  
 
Il faut alors utiliser l'URL SSH plutôt que HTTPS pour le remote :

```bash
git remote set-url origin git@github.com:lolecat/WestBaliDrivers.git
```

L'avantage du SSH : aucune ressaisie de token, l'authentification repose sur la paire de clés une fois pour toutes.

<br>

## Étape 7 : Premier push

```bash
# -u lie la branche locale à la branche distante correspondante,
# pour que les prochains "git push" n'aient plus besoin de préciser origin/main
git push -u origin main
```

Si la branche locale par défaut s'appelle `master` plutôt que `main` (dépend de la version de Git et de sa configuration), il faut adapter la commande en conséquence, ou  la renommer au préalable avec la commande suivante :

```bash
git branch -m master main
```

<br>

## Étape 8 : Vérification finale

```bash
# Confirme que le code est bien remonté
git log --oneline -5

# Récupère le hash court du commit courant, ce que
# le pipeline utilisera pour le tag sha-$GIT_SHA
git rev-parse --short HEAD
```

En rafraîchissant la page du dépôt sur GitHub, le code doit y apparaître, avec le même message de commit que celui saisi localement à l'étape 3.

<br>

## Pour la suite de la série

À partir de maintenant, chaque chapitre suppose ce dépôt existant, avec au minimum une branche `main`. Les branches `staging` (et la logique de branchement associée au pipeline CI/CD) seront créées dans le chapitre consacré à GitHub Actions.