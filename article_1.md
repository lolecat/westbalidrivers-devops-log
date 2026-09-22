# Mettre en place un pipeline CI/CD sur Azure, partie 1 : Présentation du projet

<br>

## Contexte

Ce projet est né d'un besoin concret : créer un site vitrine pour un réseau privé d'une soixantaine de chauffeurs à Bali, proposant des circuits touristiques et des transferts à la demande. Le site, baptisé **West Bali Drivers**, cible une clientèle internationale et sert avant tout de vitrine. Pas de paiement en ligne, pas de compte utilisateur, pas de back-office. L'objectif de cette première version est simple : présenter l'offre et générer des contacts via WhatsApp.

Ce que je documente ici n'est pas le développement de l'application en elle-même, mais la mise en place de l'infrastructure et du pipeline CI/CD qui permet de la livrer de façon reproductible, automatisée, et professionnelle.

<br>

## La stack technique : ce que fait chaque outil

Je ne suis pas développeur. Ce qui m'intéresse dans ce projet, c'est l'infrastructure et l'automatisation. Avant d'entrer dans le vif du sujet, voici une présentation honnête de chaque brique technique, vue depuis le prisme de quelqu'un habitué à administrer des systèmes plutôt qu'à écrire du code.

### Next.js : le moteur du site

Next.js est un framework web basé sur React (lui-même basé sur JavaScript). Pour un administrateur système, l'analogie la plus proche serait un générateur de site statique évolué : à partir de fichiers sources (templates, données, composants), il produit un ensemble de fichiers HTML, CSS et JavaScript prêts à être servis par n'importe quel serveur web.

Dans ce projet, Next.js est configuré en mode **export statique** (`output: 'export'`). Concrètement, la commande `next build` produit un dossier `out/` contenant l'intégralité du site sous forme de fichiers statiques. Pas de Node.js en production, pas de processus applicatif à surveiller, Nginx sert des fichiers, point. C'est volontairement simple pour cette phase du projet.

### TypeScript : du JavaScript avec des types

TypeScript est une surcouche de JavaScript qui ajoute le typage statique des variables et des structures de données. Si JavaScript est un langage interprété permissif (comme bash en quelque sorte), TypeScript se rapproche d'un langage compilé : les erreurs de type sont détectées à la compilation, pas à l'exécution.

Pour l'infrastructure, l'impact est indirect mais réel : un projet TypeScript se **build** avant de se déployer. Le pipeline CI/CD devra donc inclure une étape de compilation, et tout échec de build doit bloquer le déploiement.

### Tailwind CSS : le système de style

Tailwind CSS est une bibliothèque de styles visuels (couleurs, marges, polices, mises en page). Pour un non-développeur, il n'y a rien à savoir sur Tailwind d'un point de vue opérationnel, ses fichiers sont intégrés au build et n'ont aucun impact sur l'infrastructure de déploiement.

### Mapbox : les cartes interactives

Mapbox est un service externe de cartographie, comparable à Google Maps mais avec une offre gratuite plus généreuse et de meilleures options de personnalisation. Le site l'utilise pour afficher les trajets de transfert entre deux points sur une carte interactive.

Techniquement, Mapbox est consommé via une **clé d'API** (`NEXT_PUBLIC_MAPBOX_TOKEN`). C'est la seule variable sensible du projet côté application. La gestion de ce secret dans le pipeline CI/CD sera un point d'attention spécifique.

### Cloudinary : l'hébergement des images

Cloudinary est un CDN spécialisé dans la gestion d'images. Plutôt que de stocker les photos dans le repository Git ou dans un blob Azure, elles sont hébergées sur Cloudinary qui les sert à la demande avec redimensionnement et optimisation automatiques (format WebP, lazy loading, etc.).

D'un point de vue infrastructure, Cloudinary est entièrement externe, aucune configuration à gérer de notre côté. Les images sont référencées par URL dans le code.

### Docker : la containerisation

Docker est le cœur du sujet de cette série d'articles. C'est lui qui permet d'empaqueter l'application et son environnement d'exécution dans une image portable et reproductible.

Dans ce projet, le Dockerfile utilise un **build multi-stage** :

- **Stage 1 (builder)** : une image Node.js temporaire installe les dépendances et exécute le build Next.js pour produire le dossier `out/`.
- **Stage 2 (runner)** : une image Nginx Alpine minimaliste récupère uniquement le dossier `out/` et le sert via HTTP.

L'image finale ne contient donc pas Node.js, pas de code source, pas de `node_modules`. Elle ne contient que Nginx et des fichiers statiques. Résultat : une image légère (moins de 50 Mo), sans surface d'attaque inutile, identique quel que soit l'environnement où elle tourne.

### pnpm : le gestionnaire de paquets

pnpm est l'équivalent de `apt` ou `pip` pour l'écosystème JavaScript : il installe et gère les dépendances du projet. Pour l'infrastructure, son seul impact est dans le Dockerfile : c'est lui qui est invoqué pour installer les dépendances avant le build.

### ESLint + Prettier : la qualité du code

ESLint est un linter (il détecte les erreurs et mauvaises pratiques dans le code). Prettier est un formateur automatique. Ces deux outils peuvent être intégrés comme étape de vérification dans le pipeline CI/CD, avant le build, pour bloquer un déploiement si le code ne respecte pas les standards définis.

<br>

## Ce que ce projet va nous apprendre

L'application en elle-même est volontairement simple: un site statique sans backend. C'est un choix délibéré, car l'objectif est de se concentrer sur la **rigueur du pipeline** plutôt que sur la complexité applicative. D'ailleurs, le site en lui même est dans un 1er temps simple au possible, site vitrine sans logique backend et composé de fichiers statiques. On commence petit pour faire quelque chose de prorpe, et on brode autour ensuite :)  

Un site statique containerisé, c'est le cas idéal pour se concentrer sur ce qui compte ici :

- Écrire un Dockerfile propre et optimisé, à la fois pour le développement (hot reload) et pour la production (build statique)
- Publier des images sur un registre privé (Azure Container Registry)
- Déployer automatiquement sur plusieurs environnements, avec une stratégie adaptée à chacun
- Gérer l'authentification entre GitHub Actions et Azure sans secrets statiques (OIDC)
- Configurer un domaine custom avec HTTPS
- Arbitrer consciemment entre cohérence technique et coût d'hébergement, un exercice aussi formateur que l'automatisation elle-même

### Pourquoi trois environnements aussi différents les uns des autres

Un point mérite d'être explicité avant même de rentrer dans l'architecture : les trois environnements de ce projet ne reposent pas sur la même technologie de déploiement, et c'est un choix assumé. Je préfère le documenter ici plutôt que de le laisser découvrir en cours de route.

- **Dev** reste strictement local. Next.js, en mode développement, a besoin de monter le code source en volume pour profiter du hot reload (rechargement instantané à chaque modification). Un tel montage n'a de sens que sur la machine où le code est écrit, il est pénible de le reproduire proprement sur un hôte distant. Cet environnement ne sort donc jamais de mon poste.

- **Staging** est le premier environnement distant, celui qui simule des conditions réelles. Il fait tourner l'image de production (le stage `runner` du Dockerfile, Nginx + fichiers statiques) sur **Azure Container Apps**. C'est ici que le pipeline CI/CD, le registre de containers et le déploiement automatisé prennent tout leur sens.

- **Prod** rompt volontairement avec le monde des containers. Le site final est déployé sur **Azure Static Web Apps**, un service entièrement managé qui sert directement le dossier `out/` généré par Next.js, sans Docker, sans Nginx à configurer, avec HTTPS et domaine custom inclus gratuitement.

Pourquoi cette rupture ? Par pragmatisme économique et technique. Pour un site 100% statique sans logique serveur, faire tourner un container 24h/24 pour au final ne faire que servir des fichiers HTML est une dépense et une complexité que le besoin ne justifie pas. Azure Static Web Apps fait exactement ce travail, gratuitement, avec un CDN mondial en prime.

Mais alors pourquoi passer par des containers en staging, pour les abandonner ensuite ? Ce choix a un coût que j'assume et documente : ce qui est validé en staging (un container Nginx) n'est plus strictement identique à ce qui tourne en prod (l'infrastructure propriétaire de Static Web Apps). C'est une entorse au principe "ce qui marche en local marche en prod" posé plus haut, mais une entorse volontaire, réversible, et cohérente avec la philosophie *"small is beautiful"* du projet : ne pas complexifier l'infrastructure au-delà du besoin réel de la Phase 0.

Cette rupture n'est d'ailleurs pas définitive. Dès que la Phase 1 introduira une API, une base de données et un backend applicatif, Static Web Apps ne suffira plus, et la production migrera vers Azure Container Apps : le même service que celui déjà validé en staging. Le pipeline conteneurisé construit dès maintenant est donc la fondation sur laquelle l'application se reposera lorsque le besoin de complexité applicative se fera sentir.

<br>

## Architecture cible

Voici ce que nous allons construire au fil de cette série :

```
Poste local
        │
        ├── Branche dev ──────→ Dockerfile, stage "dev" (hot reload, volume monté)
        │                       Reste strictement local, jamais déployé à distance
        │
        ▼
GitHub Repository
        │
        ▼
GitHub Actions (CI/CD)
        │
        ├── Branche staging ──→ Dockerfile, stage "runner" (Nginx + build statique)
        │                                │
        │                                └──→ Azure Container Registry (tag: staging)
        │                                                │
        │                                                └──→ Azure Container Apps
        │                                                     (conditions réelles, scale-to-zero)
        │
        └── Branche main ─────→ Next.js build (out/), sans container
                                         │
                                         └──→ Azure Static Web Apps
                                                    │
                                                    └── domaine custom + HTTPS (inclus, gratuit)
```

Trois environnements, trois logiques de déploiement différentes. Comme expliqué plus haut, il s'agit d'un choix délibéré.

<br>

## Suite

Dans le prochain chapitre, nous allons commencer par le commencement : **écrire le Dockerfile**, dans ses deux variantes (développement avec hot reload, production avec build statique), et valider que chaque stage se build et se lance correctement en local.