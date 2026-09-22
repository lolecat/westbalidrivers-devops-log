# Mettre en place un pipeline CI/CD sur Azure

<br>

## De quoi ça parle

Ce dépôt documente, sous forme d'articles, la mise en place d'un pipeline CI/CD complet sur Azure pour **West Bali Drivers**, un site vitrine pour un réseau de chauffeurs à Bali. Le site lui-même (Next.js, statique, sans backend) n'est qu'un prétexte : ce qui est documenté ici, c'est l'infrastructure et l'automatisation qui permettent de le livrer de façon reproductible, sécurisée et professionnelle.

Le fil conducteur est volontairement pédagogique et transparent : chaque brique est d'abord validée manuellement (build local, push manuel, création manuelle des ressources Azure) avant d'être automatisée dans un pipeline GitHub Actions. Les choix, les impasses et les erreurs de parcours sont documentés au même titre que les solutions retenues.

Je fonctionne aujourd'hui selon les principes DevOps mais ne suis pas développeur : je viens du monde systèmes et réseaux. Cette série s'adresse donc naturellement à un public familier de l'infra plutôt qu'à des développeurs pur jus, avec des analogies système à chaque nouvelle brique.

<br>

## Architecture cible

Le pipeline s'articule autour de trois environnements, qui reposent volontairement sur trois logiques de déploiement différentes plutôt qu'une seule stack uniforme.

Spoiler, l'ensemble ressemblera à ça :

![Schéma du pipeline de déploiement](./.draws/deployment_pipeline.drawio.png)

Mais ne fuyez pas : tout cela fera sens au gré de la lecture. Un schéma en apparence compliqué, pour un principe finalement assez simple.

Pourquoi cette rupture volontaire entre un staging conteneurisé et une prod qui ne l'est pas ? La réponse est détaillée dans l'[article 1](./article_1.md).  
En résumé : faire tourner un container 24h/24 pour servir des fichiers HTML statiques est une dépense que le besoin ne justifie pas, alors qu'Azure Static Web Apps fait exactement ce travail gratuitement. Le pipeline conteneurisé reste néanmoins la fondation sur laquelle la prod basculera dès que l'application gagnera un backend.

<br>

## Les articles

### [Partie 1 : Présentation du projet](./article_1.md)
Le point de départ de la série : pourquoi ce projet, quelle stack technique (Next.js en export statique, TypeScript, Tailwind, Mapbox, Cloudinary, Docker, pnpm), et surtout pourquoi les trois environnements dev / staging / prod reposent sur trois logiques de déploiement radicalement différentes. Un article de contexte, indispensable pour comprendre les arbitrages des chapitres suivants.

### [Partie 2 : Containerisation](./article_2.md)
Écriture du Dockerfile multi-stage (dev, builder, runner), du `docker-compose.yml` pour l'environnement de développement avec hot reload, de la configuration Nginx, et validation complète en local avant de toucher à Azure. Le cœur technique : pourquoi et comment containeriser un site qui n'a, en apparence, pas besoin de l'être.

### [Partie 3 : Azure Container Registry](./article_3.md)
Création d'un registre de containers privé sur Azure, stratégie de tagging (`staging` + SHA de commit pour le rollback), et premier push manuel de l'image pour valider la chaîne de bout en bout avant automatisation.
*Annexe : [initialiser le dépôt Git et le connecter à GitHub](./article_3_annexe.md).*

### [Partie 4 : Azure Container Apps](./article_4.md)
Déploiement de l'image sur un environnement de staging serverless avec scale-to-zero : identité managée et permission `AcrPull` sans mot de passe, création de l'environnement Container Apps, et les quelques surprises de l'outillage PowerShell Azure au passage.
*Annexe : [restreindre l'accès avec Entra ID (Easy Auth)](./article_4_annexe.md).*

### [Partie 5 : GitHub Actions pour le staging](./article_5.md)
Le grand assemblage : fédération d'identité OIDC entre GitHub et Azure (zéro secret statique), puis écriture du workflow qui build, pousse vers ACR et déploie automatiquement sur Container Apps à chaque push sur `staging`.

<br>

## À venir

- **Partie 6** : Azure Static Web Apps et le workflow de déploiement de la production (branche `main`), qui rompt volontairement avec la logique container des chapitres précédents.

<br>

## Une note sur le "small is beautiful"

Cette série ne cherche pas l'infrastructure la plus sophistiquée possible, mais celle qui correspond exactement au besoin réel du projet à l'instant T, quitte à l'assumer explicitement quand elle est imparfaite (cf. la rupture staging/prod de l'article 1). C'est aussi ce qui en fait, je l'espère, un contenu utile au-delà de ce projet précis.
