# Mettre en place un pipeline CI/CD sur Azure, partie 2 : Containerisation

<br>

## Pourquoi containeriser un site statique ?

La question est légitime. Un site statique, c'est un dossier de fichiers HTML, CSS et JavaScript. N'importe quel serveur web peut le servir. Alors pourquoi passer par Docker ?

La réponse tient en deux mots : **reproductibilité** et **anticipation**

**Reproductibilité ?**  
Sans container, le comportement du site dépend de l'environnement qui le fait tourner : version de Nginx, configuration système, variables d'environnement présentes ou absentes sur la machine. Ce qui fonctionne sur un poste de développement peut se comporter différemment sur le serveur de staging, et encore différemment en production (le désormais fameux : "mais ça marche sur ma machine !).  
Avec un container, on inverse le problème : c'est l'environnement qui est empaqueté dans l'image. La même image, validée en local, est celle qui tourne ensuite sur Azure Container Apps en staging. Ce qui est testé sur mon poste est exactement ce qui sera déployé sur cet environnement distant.  
*(La production, elle, rompt volontairement avec cette logique de container, j'explique pourquoi dans l'article précédent, section "Pourquoi trois environnements aussi différents les uns des autres".)*

**Anticipation ?**  
Ce site statique n'est qu'une première version, la Phase 0 d'une application qui évoluera vers une plateforme complète avec API, base de données, dashboard chauffeur, système de paiement. Autant de composants qui tourneront, eux aussi, dans des containers.  
Containeriser dès maintenant, même pour quelque chose d'aussi simple qu'un Nginx servant des fichiers statiques, c'est poser d'emblée les bases d'une architecture n-tier cloud-native : chaque composant applicatif dans son propre container, indépendant, remplaçable, scalable individuellement. Le pipeline CI/CD que nous allons construire autour de cette image sera réutilisé et étendu pour les phases suivantes, sans repartir de zéro.  
C'est l'un des principes fondamentaux du DevOps : les bonnes habitudes d'infrastructure se prennent au début du projet, et non quand la complexité a déjà rendu le changement coûteux.

<br>

## Le Dockerfile multi-stage

Un Dockerfile est un fichier de recette : il décrit, étape par étape, comment construire une image Docker. Dans notre cas, cette recette comporte **trois phases distinctes**, ce qu'on appelle un **build multi-stage** : un stage `dev` pour le développement en local avec rechargement à chaud, un stage `builder` intermédiaire qui compile le site, et un stage `runner` qui produit l'image de déploiement.

L'analogie la plus parlante pour un sysadmin : imaginez un build RPM. Vous avez besoin d'un environnement de compilation complet pour construire le paquet : compilateurs, headers, bibliothèques de développement. Mais le paquet final installé sur les serveurs de production ne contient que les binaires et fichiers nécessaires à l'exécution, pas l'outillage de compilation. Le multi-stage Docker fonctionne sur ce même principe, à la différence que nous avons ici un stage supplémentaire dédié entièrement au confort de développement, qui ne sera jamais déployé nulle part.

Le fichier vit à `docker/Dockerfile`. Voici son contenu complet, annoté stage par stage :

```dockerfile
# =============================================================================
# Stage 1 : dev
# Rôle : environnement de développement local avec rechargement à chaud
# Le code source n'est jamais copié dans l'image : il est monté en volume
# depuis la machine hôte au démarrage du container (voir docker-compose.yml)
# =============================================================================

FROM node:22-alpine AS dev
WORKDIR /app
EXPOSE 3000
CMD ["sh", "-c", "corepack enable && corepack prepare pnpm@11.1.2 --activate && pnpm install --frozen-lockfile && pnpm dev"]

# Notez l'absence de COPY et de RUN pnpm install classique : tout se passe
# dans le CMD, au démarrage du container plutôt qu'à la construction de l'image.
# C'est délibéré (et détaillé dans la section suivante sur docker-compose) :
# ce stage n'a aucune valeur en dehors d'un bind mount du code source.
# Le construire seul, sans volume monté, démarrerait un container avec
# un /app vide qui échouerait au pnpm install.


# =============================================================================
# Stage 2 : builder
# Rôle : installer les dépendances et compiler le site statique
# Cette image intermédiaire sera complètement écartée dans l'image finale
# =============================================================================

FROM node:22.22-alpine AS builder
ENV CI=true
# CI=true est une convention respectée par de nombreux outils JS (dont Next.js) :
# certains comportements interactifs ou d'auto-détection sont désactivés,
# pour garantir un comportement strictement identique à chaque build.

WORKDIR /app
# Définit le répertoire de travail dans le container.
# Toutes les commandes suivantes s'exécuteront depuis /app.

COPY package.json pnpm-lock.yaml ./
# On copie d'abord uniquement les fichiers de déclaration des dépendances.
# Pourquoi pas tout copier d'un coup ? Optimisation du cache Docker :
# si package.json n'a pas changé, Docker réutilise le cache de l'étape
# suivante (pnpm install) sans la relancer. Un gain de temps significatif.

RUN corepack enable && corepack prepare pnpm@11.1.2 --activate
# corepack est le gestionnaire de gestionnaires de paquets Node.js.
# Il permet d'utiliser pnpm à une version figée, sans l'installer globalement.

RUN pnpm install --frozen-lockfile --ignore-scripts
# Installe les dépendances déclarées dans pnpm-lock.yaml.
# --frozen-lockfile : interdit toute modification du lockfile pendant l'install.
# Garantit que le build est reproductible : on installe exactement
# ce qui a été validé par le développeur, pas une version plus récente.
# --ignore-scripts : n'exécute aucun script post-install de dépendances tierces.
# Bonne pratique de sécurité : un package compromis ne peut pas exécuter
# de code arbitraire pendant le build.

COPY . .
# Maintenant on copie le reste du code source.
# Les fichiers listés dans .dockerignore sont automatiquement exclus.

RUN pnpm build
# Lance la compilation Next.js.
# Produit le dossier out/ contenant l'intégralité du site statique.
# Si le build échoue (erreur TypeScript, import manquant...), 
# le Dockerfile s'arrête ici et l'image ne sera pas créée.


# =============================================================================
# Stage 3 : runner
# Rôle : servir les fichiers statiques via Nginx
# C'est cette image qui sera déployée sur Azure Container Apps
# =============================================================================

FROM nginx:1.30-alpine AS runner
# Image Nginx officielle, variante Alpine.
# Elle ne contient pas Node.js, c'est voulu.
# Taille de base : ~8 Mo.

COPY --from=builder /app/out /usr/share/nginx/html
# On récupère uniquement le dossier out/ produit par le stage builder.
# Tout le reste (node_modules, code source, outils de build) est abandonné.
# --from=builder référence l'alias défini au 2e FROM.

COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
# On écrase la configuration Nginx par défaut avec la nôtre.
# Voir la section suivante pour le détail de cette configuration.

EXPOSE 80
# Expose le container sur le port 80 au lancement du container.

CMD ["nginx", "-g", "daemon off;"]
# Lance Nginx en foreground (daemon off).
# Docker s'attend à ce que le processus principal reste en premier plan.
# S'il se termine, le container s'arrête. "daemon off" garantit ce comportement.
```

Un point important à retenir pour la suite : **seul le stage `runner` est autonome**. Le code y est figé dans l'image au moment du build (`COPY . .` dans le stage `builder`, puis les fichiers statiques recopiés dans `runner`), il n'a besoin d'aucune source externe pour fonctionner. C'est le seul des trois stages qu'on pousse vers un registre et qu'on déploie sur Azure. Le stage `dev`, à l'inverse, ne contient intrinsèquement rien : il dépend entièrement d'un volume monté au runtime, ce qui en fait un stage strictement local.

<br>

## Le docker-compose : orchestrer le stage dev

Le stage `dev` du Dockerfile, à lui seul, ne suffit pas à démarrer un environnement de développement fonctionnel, il faut encore lui monter le code source en volume, exposer le bon port, et passer quelques variables d'environnement. Plutôt que de retaper une longue commande `docker run` à chaque fois, cette configuration est déclarée une fois pour toutes dans un fichier `docker-compose.yml`, à la racine du projet :

```yaml
services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: dev
      network: host
    ports:
      - '3000:3000'
    volumes:
      - .:/app
      - /app/node_modules
      - /app/.next
    dns:
      - 8.8.8.8
      - 8.8.4.4
    environment:
      - WATCHPACK_POLLING=true
```

Pour un sysadmin, `docker-compose.yml` est l'équivalent d'un fichier d'unité systemd un peu enrichi : il décrit déclarativement comment un service doit être construit et lancé, plutôt que de le documenter sous forme de commande à copier-coller dans un README. Rien ici n'est obligatoire, tout pourrait se traduire en `docker build` + `docker run` avec les bons flags. Mais c'est versionné et ça se résume à une seule commande : `docker compose up`.

### "target: dev" : ne construire qu'un seul stage

```yaml
target: dev
```

Rappel utile : notre Dockerfile contient trois stages (`dev`, `builder`, `runner`). Sans précision, Docker construirait le dernier stage du fichier (`runner`) par défaut. `target: dev` force Docker Compose à s'arrêter au premier stage - celui qui nous intéresse ici - sans même tenter de construire les stages suivants.

### Les volumes : le point qui mérite le plus d'attention

```yaml
volumes:
  - .:/app               # (1) bind mount
  - /app/node_modules     # (2) volume anonyme
  - /app/.next            # (3) volume anonyme
```

**(1) `.:/app` : le bind mount.** Tout le répertoire du projet, sur la machine hôte, est monté dans `/app` à l'intérieur du container. C'est ce qui rend le hot reload possible : modifier un fichier `src/...` sur la machine le rend immédiatement visible dans le container, comme si le code y vivait directement. C'est l'exact inverse de la logique du stage `runner`, où le code est figé dans l'image au moment du build.

**(2) et (3) : les volumes anonymes.** Leur rôle est plus subtil : ils **masquent** le contenu de `node_modules` et `.next` qui proviendrait autrement du bind mount `(1)`. Sans ces deux lignes, le `node_modules` de la machine hôte (souvent absent, ou compilé pour un système et une architecture différents de l'Alpine Linux du container) écraserait celui installé à l'intérieur du container, et inversement.

Docker superpose les volumes dans l'ordre déclaré : le bind mount `.:/app` monte d'abord tout le répertoire, puis les volumes anonymes viennent se remonter par-dessus, précisément sur ces deux sous-chemins. Cela crée des volumes Docker séparés, gérés uniquement par le container, jamais synchronisés avec l'hôte.  

Concrètement :
- `node_modules` est installé à l'intérieur du container (au démarrage, via le `CMD` du stage `dev`) et reste binaire-compatible avec l'Alpine Linux du container, peu importe le système d'exploitation de la machine hôte
- `.next` (cache de build Next.js) reste lui aussi isolé, ce qui évite les conflits entre un cache généré côté hôte et un cache généré côté container

Quand une nouvelle dépendance est ajoutée au projet, ces volumes anonymes contiennent une version périmée de `node_modules`. Il faut alors les détruire explicitement pour forcer une réinstallation propre : `docker compose down -v && docker compose up -d` (le `-v` supprime les volumes anonymes associés au service).

### "dns" et "environment"

```yaml
dns:
  - 8.8.8.8
  - 8.8.4.4
environment:
  - WATCHPACK_POLLING=true
```

Ces deux réglages sont des palliatifs à des particularités de WSL2 (l'environnement Windows/Linux utilisé pour développer ce projet), pas des choix génériques à reproduire tels quels sur n'importe quelle machine :

- `dns` force le container à utiliser les résolveurs DNS de Google plutôt que ceux, parfois capricieux, propagés automatiquement par WSL2
- `WATCHPACK_POLLING=true` force Next.js à surveiller les fichiers par scrutation périodique (polling) plutôt que par événements `inotify` natifs, car ces événements filesystem ne traversent pas toujours proprement un bind mount sous WSL2

### Une frontière à garder en tête

Le stage `dev` et son bind mount n'ont de sens que sur la machine où le code est écrit. Il n'y a pas de "disque local" à monter côté Azure : un container Azure Container Apps démarré depuis ce stage se retrouverait avec un `/app` vide, et planterait immédiatement. C'est précisément pour cette raison que seul le stage `runner` (autonome, code figé dans l'image) est candidat au déploiement distant. Le stage `dev` reste, par construction, un outil strictement local.

<br>

## La configuration Nginx

La configuration par défaut de Nginx est suffisante pour servir des fichiers statiques simples, mais elle présente quelques lacunes pour un site Next.js en production. Voici notre configuration personnalisée :

```nginx
# docker/nginx.conf

server {
    listen 80;
    server_name _;
    # server_name _ : accepte les requêtes pour n'importe quel nom de domaine.
    # Derrière Azure Container Apps, c'est Azure qui gère le nom de domaine
    # généré et forward les requêtes au container : ce wildcard est appropriée.

    root /usr/share/nginx/html;
    # Répertoire racine où Nginx cherche les fichiers à servir.
    # C'est là où nous avons copié le contenu de out/.

    index index.html;

    # Gestion du routing côté client
    location / {
        try_files $uri $uri/ $uri.html /index.html;
        # Next.js en export statique génère des fichiers .html pour chaque route.
        # Ex : /tours/ubud-culture-nature → out/tours/ubud-culture-nature.html
        # try_files essaie dans l'ordre :
        #   1. Le fichier exact ($uri)
        #   2. Un dossier avec index ($uri/)
        #   3. Le fichier .html correspondant ($uri.html)
        #   4. Fallback sur index.html (pour les routes dynamiques éventuelles)
        # Sans cette directive, Nginx retournerait 404 sur les URLs directes.
    }

    # Cache long terme pour les assets statiques
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        # Les assets générés par Next.js ont des noms avec hash de contenu
        # (ex: _next/static/chunks/main-abc123.js).
        # Un hash différent = un fichier différent = pas de problème de cache.
        # On peut donc agressivement mettre en cache pour 1 an.
        # "immutable" indique au navigateur de ne jamais revalider.
    }

    # Compression gzip
    gzip on;
    gzip_types text/plain text/css application/javascript application/json
               image/svg+xml;
    gzip_min_length 1024;
    # Active la compression pour les types de fichiers textuels.
    # Réduit la taille des transferts réseau de 60 à 80% pour le JS/CSS.
    # gzip_min_length : ne compresse pas les fichiers de moins de 1 Ko
    # (le surcoût CPU n'en vaut pas la peine pour les petits fichiers).

    # Page d'erreur 404
    error_page 404 /404.html;
    # Next.js génère automatiquement un fichier 404.html lors du build.
}
```

<br>

## Le `.dockerignore`

Le `.dockerignore` fonctionne exactement comme un `.gitignore` : il liste les fichiers et dossiers à exclure lors de la copie vers le contexte de build Docker. Sans lui, la commande `COPY . .` dans le Dockerfile enverrait l'intégralité du répertoire de travail au daemon Docker, y compris des fichiers volumineux ou sensibles qui n'ont rien à faire dans l'image.

```bash
# Dépendances. Réinstallées dans le container via pnpm install.
node_modules/

# Dossier de build Next.js (dev). Le build de prod se fait dans le container.
.next/

# Output statique précédent. Généré dans le container.
out/

# Fichiers d'environnement. Ne jamais inclure dans une image Docker
.env.local
.env*.local

# Documentation du projet, inutile dans l'image.
*.md

# Configuration IDE et outils locaux
.vscode/

# Git
.git/
.gitignore

# Docker lui-même, pas besoin de s'inclure
Dockerfile*
docker-compose*.yml
.dockerignore

# Fichiers système
.DS_Store
Thumbs.db
```

Les bénéfices sont doubles : **performance** (le contexte de build transmis au daemon est plus petit, donc plus rapide) et **sécurité** (on s'assure qu'aucun fichier sensible comme `.env.local` ne se retrouve dans l'image par inadvertance).

<br>

## Build et test en local

Avant de pousser quoi que ce soit vers Azure, on valide que l'image se construit et se comporte correctement en local.

### Construire l'image

```bash
docker build \
  -f docker/Dockerfile \
  --target runner \
  --build-arg NEXT_PUBLIC_MAPBOX_TOKEN=le_token \
  -t westbalidrivers:local \
  .
```

- `-f docker/Dockerfile` : spécifie le Dockerfile à utiliser. C'est le même fichier que celui utilisé par `docker compose up`, mais on cible ici un autre stage
- `--target runner` : construit uniquement le stage `runner` (et son prérequis `builder`), en ignorant complètement le stage `dev`
- `-t westbalidrivers:local` : nomme et tague l'image
- `.` : le contexte de build est le répertoire courant

Le build prend quelques minutes la première fois (téléchargement des images de base, installation des dépendances). Les relances suivantes sont quasi instantanées grâce au cache Docker, sauf si `package.json` a changé.

### Vérifier la taille de l'image

```bash
docker images westbalidrivers:local
```

```
IMAGE                   ID             DISK USAGE   CONTENT SIZE   EXTRA
westbalidrivers:local   eebfe3ebd8b1       98.9MB         27.3MB 
```

- Disk Usage : l'espace réellement occupé sur le disque une fois les layers décompressés et extraits en fichiers, plus une part de métadonnées. Le contenu ("content") représente la taille de l'image avant décompression, ce qui explique pourquoi la consommation disque est plus importante une fois inspectée  
- Content size : la taille des layers de l'image telle que stockée/transférée, c'est-à-dire compressée. C'est ce qui compte pour un docker push/pull : la quantité de données à transférer sur le réseau.


### Lancer le container et tester

```bash
docker run \
  --rm \
  -p 8080:80 \
  --name westbalidrivers-test \
  westbalidrivers:local
```

- `--rm` : supprime le container automatiquement à l'arrêt
- `-p 8080:80` : publie le port 80 du container sur le port 8080 de la machine hôte
- `--name` : donne un nom au container pour faciliter les commandes suivantes

Le site est maintenant accessible sur `http://localhost:8080`. On vérifie :

```bash
# Vérification basique : doit retourner HTTP 200
curl -I http://localhost:8080

# Vérification du routing : doit retourner 200, pas 404
curl -I http://localhost:8080/tours

# Vérification des headers de cache sur un asset statique
curl -I http://localhost:8080/_next/static/chunks/main.js
# On doit voir : Cache-Control: public, immutable
#                Content-Encoding: gzip
```

### Inspecter le contenu de l'image

```bash
# Ouvrir un shell dans le container pour vérifier son contenu
docker run --rm -it --entrypoint sh westbalidrivers:local

# Dans le container :
ls /usr/share/nginx/html        # doit contenir index.html, _next/, tours/, etc.
cat /etc/nginx/conf.d/default.conf  # vérifier que notre config est bien en place
nginx -t                        # valider la syntaxe de la configuration Nginx
```

### Checklist avant de passer à la suite

Avant de commencer à configurer Azure, s'assurer que :

- [x] `docker build` se termine sans erreur
- [x] L'image fait moins de 100 Mo
- [x] `http://localhost:8080` répond HTTP 200
- [x] Les routes du site (`/tours`, `/transfers`, `/gallery`) répondent 200
- [x] Les headers `Cache-Control` sont présents sur les assets statiques
- [x] `nginx -t` ne remonte aucune erreur de syntaxe
- [x] `.env.local` **n'est pas** présent dans l'image (vérifiable via le shell dans le container)

<br>

## Ce que nous avons accompli

À ce stade, un seul Dockerfile nous donne deux choses distinctes :

- Un environnement de développement local (stage `dev`, orchestré par `docker compose up`), avec hot reload et code monté en volume qui ne sortira jamais de la machine
- Une image de déploiement (stage `runner`) qui :
  - Pèse moins de 100 Mo
  - Ne contient que Nginx et des fichiers statiques. Aucun outil de build, aucun code source
  - Est identique quel que soit l'environnement distant où elle tournera
  - Gère correctement le routing, le cache et la compression

C'est la brique de base sur laquelle tout le pipeline va s'appuyer. Une image propre et validée localement, c'est la garantie que les problèmes que l'on rencontrera dans la suite seront des problèmes d'infrastructure et non des problèmes d'application.

<br>

## Suite

Dans le prochain chapitre, nous allons créer notre **Azure Container Registry** et y pousser cette image pour la première fois, manuellement. Ce geste manuel est important : il permet de valider la chaîne de bout en bout avant de l'automatiser avec GitHub Actions.

