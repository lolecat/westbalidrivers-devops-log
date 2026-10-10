# Mettre en place un pipeline CI/CD sur Azure, partie 6 : la production sur Azure Static Web Apps

<br>

## Où on en est, et un changement de cap

Le staging est désormais entièrement automatisé : un push sur `staging` construit l'image, la pousse dans ACR sous un tag `sha-<hash>`, et met à jour la Container App. Reste la production.

Le plan initial, annoncé dès l'article 1 et rappelé à la fin de l'article 5, était le suivant : un push sur `main` déclenche un `next build`, et le dossier `out/` produit est envoyé directement sur Azure Static Web Apps. Pas de container, pas d'ACR, pas d'identité Azure.

En préparant ce chapitre, j'ai changé d'avis. Ce plan a un défaut que je n'avais pas vraiment mesuré : **le site déployé en production n'aurait jamais été testé tel quel**. Il serait reconstruit à partir des mêmes sources, certes, mais par un autre build, sur un autre runner, à un autre moment. Les dépendances sont figées par le lockfile, donc dans 99 % des cas le résultat serait identique. Mais « dans 99 % des cas identique », ça n'est pas « identique ».

Le principe que je retiens à la place est un classique du déploiement continu : **build once, deploy many**. On construit l'artefact une seule fois, on le valide en staging, et c'est *ce même artefact* qu'on promeut en production. Concrètement :

- l'image `sha-<hash>` validée en staging est marquée comme « prête pour la prod » par un tag de version (`v0.1.0`) ;
- on extrait de cette image les fichiers statiques qu'elle contient ;
- on publie ces fichiers sur Static Web Apps.

Ce changement a trois conséquences que je trouve toutes positives :

1. **L'entorse documentée dans l'article 1 se réduit nettement.** J'y écrivais que ce qui est validé en staging (un container Nginx) n'est pas strictement ce qui tourne en production. C'est toujours vrai pour le *serveur* (Nginx d'un côté, l'infrastructure de Static Web Apps de l'autre), mais les *fichiers servis*, eux, deviennent identiques à l'octet près.
2. **Le staging devient une vraie porte d'entrée vers la production.** Pas d'image validée en staging, pas de mise en production possible.
3. **Le rollback ne demande aucun rebuild.** Revenir à la version précédente revient à repromouvoir le tag `v0.0.9`, dont l'artefact existe déjà dans ACR et a déjà tourné.

En contrepartie, la production *consomme* désormais ACR, ce que les articles 1, 3 et 5 annonçaient explicitement comme ne devant pas arriver. Je préfère assumer ce revirement ici plutôt que de réécrire l'histoire : c'est aussi ça, une architecture qui évolue.

> Petit ménage au passage : la branche de production du dépôt s'appelait encore `master`, alors que toute la série parle de `main`. Elle a été renommée avant d'écrire ce chapitre. Aucun workflow ni aucune identité ne dépendait encore d'elle, c'était donc le moment le moins coûteux pour le faire.

Ce chapitre se concentre sur la partie Azure et sur une **première promotion entièrement manuelle**, fidèle à la règle suivie depuis le début de la série : valider à la main avant d'automatiser. Le workflow GitHub Actions de production fera l'objet du chapitre suivant.

<br>

## Le prérequis de la promotion : un artefact agnostique de l'environnement

Promouvoir un artefact n'a de sens que si cet artefact ne contient **rien de spécifique à l'environnement** dans lequel il a été construit. Tout ce que Next.js inline au moment du build est figé dans l'image staging, et partira tel quel en production.

Dans notre cas, la seule valeur injectée au build est `NEXT_PUBLIC_MAPBOX_TOKEN`, et c'est le même token pour tous les environnements. Le reste de ce qui distingue le staging de la production vit déjà, ou vivra, dans la **couche d'hébergement** :

- le staging est protégé par Easy Auth (annexe de l'article 4), configuré sur la Container App, pas dans le site ;
- la production, comme on va le voir, portera ses propres en-têtes HTTP via la configuration de Static Web Apps.

La règle à garder en tête pour la suite : **ce qui varie par environnement appartient à l'hébergement, pas à l'artefact**. Le jour où le token Mapbox sera séparé entre la prod et le reste (pour le restreindre au domaine de production), il faudra soit le charger au runtime, soit renoncer à la promotion pour revenir à un rebuild.

<br>

## Étape 1 : créer la Static Web App

### Prérequis

```powershell
Register-AzResourceProvider -ProviderNamespace Microsoft.Web
```

### Variables de travail

```powershell
$rg       = "rg-westbalidrivers"
$location = "westeurope"
$swa_name = "swa-westbalidrivers-prod"
```

### Création

```powershell
New-AzStaticWebApp `
  -ResourceGroupName $rg `
  -Name $swa_name `
  -Location $location `
  -SkuName "Free"
```

Le SKU "Free" suffit largement pour la Phase 0 : HTTPS et domaine custom inclus, CDN mondial, aucun coût. Le SKU Standard apporte un SLA, plus de domaines custom et des fonctionnalités d'authentification avancées, rien dont un site vitrine statique ait besoin aujourd'hui.

La `-Location` ne détermine pas où le site est servi (le contenu est distribué mondialement), mais seulement où vivent les métadonnées de la ressource. On reste cohérent avec le reste du projet en choisissant `westeurope`.

**Point d'attention : ce qu'on ne passe pas à ce cmdlet.** `New-AzStaticWebApp` accepte des paramètres `-RepositoryUrl`, `-RepositoryToken` et `-Branch`. Les renseigner déclenche le parcours « clé en main » de Static Web Apps : Azure **commite lui-même un fichier de workflow** dans le dépôt GitHub, avec sa propre logique de build, et configure les secrets nécessaires. C'est pratique pour un premier essai, mais c'est exactement l'inverse de ce qu'on veut ici : on perdrait la maîtrise du pipeline, et ce workflow reconstruirait le site au lieu de promouvoir l'artefact validé. On crée donc une ressource « nue », que l'on alimentera nous-mêmes.

Effet de bord appréciable : sans dépôt lié, Static Web Apps ne crée pas non plus d'environnements de prévisualisation sur chaque pull request. On a déjà un staging, pas besoin d'un second.

### Récupérer l'URL générée

```powershell
$swa = Get-AzStaticWebApp -ResourceGroupName $rg -Name $swa_name
$swa.DefaultHostname
```

Le résultat ressemble à `<mots-aléatoires>-<hash>.azurestaticapps.net`. À ce stade, cette URL sert une page d'attente générique : rien n'a encore été déployé.

<br>

## Étape 2 : configuration de l'hébergement avec "staticwebapp.config.json"

En staging, c'est `docker/nginx.conf` qui décide de la façon dont les fichiers sont servis. En production, ce rôle revient à un fichier `staticwebapp.config.json`, lu par Static Web Apps au moment du déploiement.

### Où le ranger

Deux options se présentaient :

- le placer dans `public/` : Next.js le copierait dans `out/`, il se retrouverait donc dans l'image staging (inerte sous Nginx, mais servi publiquement à quiconque demande `/staticwebapp.config.json`) ;
- le versionner à part, et l'ajouter aux fichiers extraits au moment du déploiement en production.

Je retiens la seconde, pour rester fidèle à la règle posée plus haut : c'est de la configuration d'hébergement, elle n'a rien à faire dans l'artefact. Il faut donc créer un répertoire `deploy/swa` à la racine du projet, et y ajouter le fichier `staticwebapp.config.json`.

### Le contenu

```json
{
  "responseOverrides": {
    "404": {
      "rewrite": "/404.html",
      "statusCode": 404
    }
  },
  "globalHeaders": {
    "X-Robots-Tag": "noindex, nofollow",
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Content-Security-Policy-Report-Only": "default-src 'self'; script-src 'self' 'unsafe-inline' https://api.mapbox.com; style-src 'self' 'unsafe-inline' https://api.mapbox.com; img-src 'self' data: blob: https://res.cloudinary.com https://api.mapbox.com; connect-src 'self' https://api.mapbox.com https://events.mapbox.com; worker-src 'self' blob:; frame-ancestors 'none'"
  },
  "routes": [
    {
      "route": "/_next/static/*",
      "headers": {
        "Cache-Control": "public, max-age=31536000, immutable"
      }
    }
  ]
}
```

Quelques choix méritent d'être expliqués.

**L'absence de `navigationFallback`.** C'est l'équivalent Static Web Apps du dernier argument de `try_files` dans Nginx. Configuré vers `/index.html`, il renverrait la page d'accueil pour n'importe quelle URL inexistante, masquant ainsi les vraies 404. C'est exactement ce qu'on a évité côté Nginx avec `=404`. Sans `navigationFallback`, Static Web Apps ne sert que les fichiers réellement déployés : la parité avec le staging est respectée.

**`responseOverrides` sur le code 404.** Static Web Apps ne sert pas automatiquement le `404.html` généré par Next.js : il faut le lui indiquer, comme on l'a fait avec `error_page 404 /404.html` dans Nginx. Le `statusCode: 404` explicite garantit que la page d'erreur personnalisée est bien renvoyée *avec* un code 404, et non un 200. La différence est invisible pour un visiteur, mais pas pour un moteur de recherche ni pour un outil de supervision.

**`X-Robots-Tag: noindex, nofollow`, provisoire.** Tant que le domaine custom n'est pas en place, le site de production est public sur son URL `*.azurestaticapps.net`. Si un moteur de recherche l'indexe à cette adresse, il faudra ensuite gérer du contenu dupliqué entre cette URL et le futur domaine. Cet en-tête l'en empêche. Il est posé au niveau de l'hébergement, pas dans le HTML : il disparaîtra par simple modification de ce fichier, le jour où le domaine sera branché, sans toucher à l'artefact.

**La CSP en mode `Report-Only`.** Une Content Security Policy restreint les origines depuis lesquelles le navigateur accepte de charger scripts, styles, images, etc. Mal réglée, elle casse le site sans prévenir : la carte Mapbox, par exemple, a besoin de `api.mapbox.com`, de `events.mapbox.com` et de workers en `blob:`, et Next.js injecte des scripts inline. En mode `Report-Only`, le navigateur n'applique rien : il signale seulement dans sa console ce qu'il *aurait* bloqué. On observe, on ajuste, et on bascule en mode bloquant une fois la politique validée. Toujours la même philosophie : valider avant d'appliquer.

<br>

## Étape 3 : promouvoir manuellement la version actuellement en staging

On va dérouler à la main, une première fois, ce que le pipeline de production fera ensuite automatiquement.

### Identifier l'image qui tourne en staging

```powershell
$appRevision = Get-AzContainerAppRevision -ContainerAppName "ca-westbalidrivers-staging" -ResourceGroupName $rg `
    | ?{$_.Active -eq $true} `
    | Select-Object -Property Name, CreatedTime, @{Name ='Image'; Expression = {$_.TemplateContainer.Image}}
```

L'image de la révision active porte un tag de la forme `sha-95ab7cf`. C'est cette image, et elle seule, que l'on va promouvoir.

```powershell
$short_sha = $appRevision.Image.Substring($appRevision.Image.Length - 7) # Extraire les 7 derniers caractères du nom de l'image
$version   = "v0.1.0"
$acr       = "acrwestbalidrivers.azurecr.io"
$image     = "$acr/westbalidrivers"
```

### Poser le tag de version côté Git

Le tag Git marque le **commit source** de l'artefact. Il rend la correspondance entre une version en production et l'état exact du code consultable par n'importe qui ayant accès au dépôt.

```bash
git fetch origin staging
$full_sha=$(git rev-parse "95ab7cf^{commit}")
git tag -a v0.1.0 "$full_sha" -m "Première mise en production"
git push origin v0.1.0
```

Un tag *annoté* (`-a`) plutôt qu'un tag léger : il enregistre l'auteur, la date et un message. C'est une trace d'audit minimale, mais réelle, de qui a décidé de promouvoir quoi, et quand.

### Poser le même tag côté ACR

```powershell
Connect-AzContainerRegistry -Name "acrwestbalidrivers"
```

```bash
docker pull acrwestbalidrivers.azurecr.io/westbalidrivers:sha-95ab7cf
docker tag  acrwestbalidrivers.azurecr.io/westbalidrivers:sha-95ab7cf acrwestbalidrivers.azurecr.io/westbalidrivers:v0.1.0
docker push acrwestbalidrivers.azurecr.io/westbalidrivers:v0.1.0
```

Le `docker push` est quasi instantané : tous les layers sont déjà dans ACR, seul un nouveau tag est créé, pointant vers **le même manifest**. On le vérifie :

```powershell
Get-AzContainerRegistryTag -RegistryName "acrwestbalidrivers" -RepositoryName "westbalidrivers"
```

Les tags `sha-95ab7cf` et `v0.1.0` doivent apparaître sur la même image, avec le même digest.

Ce tag de version a un second rôle, qu'on exploitera quand on écrira enfin la politique de rétention laissée en suspens à l'article 3 : les tags `sha-*` pourront être purgés régulièrement, tandis que les tags `v*` seront conservés. Une image promue en production reste ainsi disponible comme point de rollback, même une fois son tag `sha-*` supprimé.

### Noter le digest

Un tag peut techniquement être déplacé vers une autre image. Un digest (`sha256:...`), lui, désigne un contenu précis et ne peut pas mentir. On le relève pour garder une trace exacte de ce qui part en production :

```bash
docker inspect --format '{{index .RepoDigests 0}}' acrwestbalidrivers.azurecr.io/westbalidrivers:v0.1.0
```

> Petite correction au passage : l'article 5 affirmait que la Container App était déployée vers « un digest précis et immuable » via le tag `sha-...`. C'était un abus de langage : `sha-95ab7cf` est un tag, certes jamais réécrit par notre pipeline, mais un tag tout de même. Le digest, c'est ce `sha256:...`.

### Extraire les fichiers statiques de l'image

On n'a pas besoin de *lancer* le container pour récupérer ses fichiers : `docker create` instancie un container à l'arrêt, dont on copie simplement le contenu.

```bash
docker create --name swa-extract acrwestbalidrivers.azurecr.io/westbalidrivers:v0.1.0
docker cp swa-extract:/usr/share/nginx/html ./release
docker rm swa-extract
```

### Ajouter la configuration d'hébergement

```bash
cp deploy/swa/staticwebapp.config.json ./release/
```

Le dossier `./release` contient maintenant exactement ce qui va partir en production : l'artefact validé en staging, plus la configuration propre à cet hébergement.

<br>

## Étape 4 : déployer avec la SWA CLI

Static Web Apps accepte les déploiements via un **token de déploiement**, propre à chaque ressource. On le récupère avec le module Az, sans jamais l'afficher, en le plaçant directement dans la variable d'environnement attendue par l'outil de déploiement :

```powershell
$env:SWA_CLI_DEPLOYMENT_TOKEN = ((Get-AzStaticWebAppSecret -ResourceGroupName $rg -Name $swa_name).Property | ConvertFrom-Json).apiKey
```

Le déploiement se fait avec la **SWA CLI**, l'outil officiel de Microsoft. Plutôt que d'installer Node.js et cet outil sur le poste, on le lance dans un container éphémère, cohérent avec l'approche de la série :

```
docker run --rm `
  -v "$(pwd)/release:/release" `
  -e SWA_CLI_DEPLOYMENT_TOKEN `
  node:22 `
  npx -y @azure/static-web-apps-cli deploy /release --env production
```

**Point d'attention sur `--env production`** : sans cette option, la SWA CLI déploie par défaut dans un environnement de **prévisualisation**, et non en production. Le déploiement réussit, la CLI affiche une URL… mais pas celle du site de production, qui continue de servir sa page d'attente. C'est le genre d'erreur qui fait perdre un bon quart d'heure, alors autant le savoir avant.

Une fois le déploiement terminé, on fait le ménage :

```powershell
Remove-Item env:SWA_CLI_DEPLOYMENT_TOKEN
```

Ce token est un secret à longue durée de vie, exactement le genre d'identifiant que l'article 5 s'est efforcé d'éliminer. Il est acceptable pour une manipulation manuelle ponctuelle, récupéré à la volée et jamais stocké, mais pas pour un pipeline. Dans le chapitre suivant, le workflow le récupérera au runtime via une identité OIDC dédiée, et il ne sera jamais enregistré dans les secrets GitHub. En cas de doute sur une fuite, il peut être régénéré depuis le portail (*Manage deployment token*), ce qui invalide immédiatement l'ancien.

<br>

## Étape 5 : validation

```bash
fqdn="mango-cliff-05bf87d03.4.azurestaticapps.net"

# Page d'accueil : 200
curl -I "https://$fqdn/"

# Routes du site : 200
curl -I "https://$fqdn/tours"
curl -I "https://$fqdn/transfers"
curl -I "https://$fqdn/gallery"

# Une URL inexistante : 404, et non 200 avec la page d'accueil
curl -I "https://$fqdn/cette-page-nexiste-pas"

# En-têtes d'hébergement : X-Robots-Tag, en-têtes de sécurité, CSP Report-Only
curl -sI "https://$fqdn/" | grep -iE "x-robots-tag|x-content-type|x-frame|referrer|content-security"

# Le fichier de configuration ne doit pas être servi publiquement
curl -I "https://$fqdn/staticwebapp.config.json"
```

Le test sur l'URL inexistante est le plus important de la liste : c'est lui qui confirme que l'absence de `navigationFallback` produit bien le même comportement que `=404` en staging.


<br>

## Ce qui reste manuel, et ne le restera pas

Cette première promotion a demandé une bonne dizaine de commandes, dans deux shells, avec un secret manipulé à la main. C'était voulu, pour que chaque étape soit validée isolément. Le chapitre suivant automatise l'ensemble :

- **un workflow `production.yml`**, déclenché manuellement (`workflow_dispatch`) avec le tag à promouvoir en paramètre
- **une identité dédiée à la production**, distincte de celle du staging, avec une "federated credential" restreinte à la branche `main` et des droits limités au strict nécessaire
- **un garde-fou structurel** : le workflow refusera de promouvoir un tag dont l'image `sha-*` n'existe pas dans ACR
- **la politique de rétention ACR**, enfin, conçue pour conserver les versions promues

<br>

## Checklist avant de passer à la suite

- [x] La Static Web App `swa-westbalidrivers-prod` existe en SKU Free, **sans** dépôt GitHub lié
- [x] `deploy/swa/staticwebapp.config.json` est versionné dans le dépôt, hors de `public/`
- [x] Le commit promu porte le tag Git annoté `v0.1.0`
- [x] L'image correspondante porte les tags `sha-<hash>` et `v0.1.0` dans ACR, sur le même digest, et ce digest est noté
- [x] Le dossier extrait ne contient plus `50x.html`
- [x] Le déploiement a été fait avec `--env production`, et le token a été retiré de la session
- [x] Le site répond en HTTPS sur son URL `azurestaticapps.net`, routes comprises
- [x] Une URL inexistante répond 404 avec la page personnalisée
- [x] Les en-têtes `X-Robots-Tag`, de sécurité et de cache sont présents
- [x] `staticwebapp.config.json` n'est pas accessible publiquement
- [x] La carte Mapbox fonctionne en production

<br>

## Suite

La production tourne, et elle sert exactement les fichiers validés en staging. Il ne reste plus qu'à retirer l'humain des étapes mécaniques sans le retirer de la décision. Dans le prochain chapitre, on écrit le workflow `production.yml` : promotion d'un tag en un clic, identité OIDC dédiée, aucun secret stocké, et un rollback qui se résume à relancer le workflow avec le tag précédent.