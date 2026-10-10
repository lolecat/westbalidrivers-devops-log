# Mettre en place un pipeline CI/CD sur Azure, partie 3 : Azure Container Registry

<br>

## Qu'est-ce qu'un registre de containers ?

Un registre de containers est un entrepôt centralisé pour stocker et distribuer des images Docker. Si Docker Hub est l'équivalent public (comme GitHub pour le code), **Azure Container Registry (ACR)** est son équivalent privé hébergé dans votre tenant Azure.

L'analogie avec un dépôt de paquets RPM ou APT tient bien : on y publie des artefacts versionnés, on les récupère depuis n'importe quel environnement autorisé, et on garde un historique des versions. La différence, c'est que l'artefact n'est pas un paquet mais une image Docker complète : l'application et son environnement d'exécution réunis.

Dans notre pipeline, ACR joue le rôle de **point de vérité unique** pour les images vouées à un déploiement conteneurisé :
- GitHub Actions y pousse une image après chaque build de la branche `staging`
- Azure Container Apps la récupère depuis ACR au déploiement
- Personne ne construit une image directement sur un serveur de destination, on déploie toujours depuis le registre

Un point à garder en tête : ACR n'intervient que pour l'environnement staging. La branche `dev` reste strictement locale (voir l'article 1), et la production, elle, ne consomme jamais d'image. Azure Static Web Apps déploie directement le dossier `out/` généré par `next build`, sans passer par Docker ni par un registre. ACR n'est donc pas le point de passage universel de tout le pipeline, mais celui de sa seule étape conteneurisée.

<br>

## Pourquoi ACR plutôt que Docker Hub ?

Docker Hub est public par défaut et le passage en privé est payant au-delà d'un repository. Surtout, pour un déploiement Azure, ACR offre une intégration native :

- **Authentification transparente** : les services Azure (Container Apps, AKS) peuvent se connecter à ACR via des identités managées, sans mot de passe à gérer
- **Réseau privé** : le trafic entre ACR et les services Azure peut rester sur le réseau interne Azure, sans transiter par Internet
- **Géolocalisation** : ACR est déployé dans la même région que vos autres ressources, ce qui garantit une latence minime au pull
- **Rétention et nettoyage** : des politiques de rétention permettent de supprimer automatiquement les vieilles images pour maîtriser les coûts de stockage

<br>

## Création de l'ACR

Toutes les opérations qui suivent se feront via Powerhshell. C'est une choix purement personnel, l'AZ CLI fait tout aussi bien le travail.  
Les commandes Docker, elles, restent en bash puisqu'elles s'exécutent dans mon environnement WSL2 habituel, les deux contextes cohabitent naturellement sur une machine Windows.


### Prérequis

```powershell
# Vérifier que le module Az est installé et à jour
Get-InstalledModule -Name Az

# Se connecter à Azure (ouvre une fenêtre de connexion interactive)
Connect-AzAccount

# Vérifier qu'on est sur le bon abonnement
Get-AzContext | Select-Object Name, Subscription

# Si nécessaire, basculer sur le bon abonnement, ici "Root subscription"
Set-AzContext -Subscription "Root subscription"

# Enregistrer le resource provider ACR
Register-AzResourceProvider -ProviderNamespace Microsoft.ContainerRegistry
```

### Variables de travail

```powershell
$rg       = "rg-westbalidrivers"
$location = "westeurope"
$acr_name = "acrwestbalidrivers"         # doit être unique globalement, sans tirets ni majuscules
$sku      = "Basic"                      # Basic suffit amplement pour un projet de ce type
```

**Note sur le SKU :** ACR existe en trois niveaux : Basic, Standard, Premium. Basic offre 10 Go de stockage et suffit largement pour un projet solo ou une petite équipe. Standard (100 Go, geo-réplication possible) et Premium (réplication multi-régions, réseau privé) sont adaptés aux projets d'entreprise. Le coût du Basic est légèrement inférieur à 5€/mois.


### Création du Resource Group (si non existant)

```powershell
New-AzResourceGroup -Name $rg -Location $location
```

### Création de l'ACR

```powershell
$registry = New-AzContainerRegistry -ResourceGroupName $rg -Name $acr_name -Sku $sku

# Le compte admin (login/mot de passe statique) reste désactivé par défaut.
# On s'authentifiera via Connect-AzContainerRegistry, adossé à une identité Azure,
# plutôt que via un couple identifiant/mot de passe fixe.
```

Vérification :
```powershell
Get-AzContainerRegistry -ResourceGroupName $rg -Name $acr_name | Select-Object Name, LoginServer, @{Name = 'Sku'; Expression = {$_.SkuTier}}
```

```
Name                  LoginServer                          Sku
--------------------  -----------------------------------  -----
acrwestbalidrivers    acrwestbalidrivers.azurecr.io        Basic
```
L'adresse `acrwestbalidrivers.azurecr.io` est le endpoint depuis lequel Docker pourra pousser et récupérer des images.

<br>

## Stratégie de tagging

Avant de pousser notre première image, il faut définir une convention de nommage. Les tags Docker sont l'équivalent des versions de paquets, ils permettent d'identifier précisément quelle image est déployée où.

Ici, la convention est plus simple qu'elle n'aurait pu l'être, précisément parce qu'un seul environnement consomme des images depuis ACR :

| Tag | Déclenché par | Déployé sur |
|---|---|---|
| `staging` | Push sur la branche `staging` | Azure Container Apps |
| `sha-abc1234` | Chaque build CI sur `staging` | Conservé dans ACR pour rollback |

Les branches `dev` et `main` ne poussent jamais rien vers ACR : `dev` reste locale, et `main` déclenche un déploiement direct vers Azure Static Web Apps à partir du dossier `out/`, sans étape Docker.

Le tag par SHA de commit reste important même avec un seul environnement : il permet de retrouver et redéployer une version exacte de l'image, indépendamment du tag mouvant `staging` qui, lui, pointe toujours vers le dernier build. C'est votre filet de sécurité pour les rollbacks.

Le format complet d'une référence d'image dans notre ACR sera :

```
acrwestbalidrivers.azurecr.io/westbalidrivers:staging
acrwestbalidrivers.azurecr.io/westbalidrivers:sha-abc1234
```

<br>

## Premier push manuel

On effectue ce premier push manuellement pour valider la chaîne de bout en bout avant de l'automatiser.  
Toujours utile de valider manuellement avant d'automatiser.

> **Rappel du chapitre précédent :** l'image doit avoir été construite avec `--build-arg NEXT_PUBLIC_MAPBOX_TOKEN=...`, sans quoi la carte Mapbox restera cassée une fois déployée. Cette variable est injectée au moment du `pnpm build`, pas lue au runtime. Si ce n'est pas déjà fait :
>
> ```bash
> docker build \
>   -f docker/Dockerfile \
>   --target runner \
>   --build-arg NEXT_PUBLIC_MAPBOX_TOKEN=le_token \
>   -t westbalidrivers:local \
>   .
> ```

### S'authentifier auprès de l'ACR

```powershell
Connect-AzContainerRegistry -Name $acr_name
# Le module Az récupère un token temporaire et configure Docker pour l'utiliser
# (ce cmdlet appelle en interne un docker login avec un token de courte durée).
# Ce token est valable 3 heures, ce qui est suffisant pour nos tests manuels.
# En CI/CD, l'authentification se fera différemment (via OIDC, chapitre 5).
```

### Tagger l'image locale pour l'ACR

```bash
# L'image construite au chapitre précédent
docker tag westbalidrivers:local acrwestbalidrivers.azurecr.io/westbalidrivers:staging

# On crée également le tag SHA pour simuler ce que fera le pipeline (attention, le shell courant est toujours Powershell)
$git_sha=$(git rev-parse --short HEAD)
docker tag westbalidrivers:local acrwestbalidrivers.azurecr.io/westbalidrivers:sha-$git_sha
```

### Pousser vers ACR

```bash
docker push acrwestbalidrivers.azurecr.io/westbalidrivers:staging
docker push acrwestbalidrivers.azurecr.io/westbalidrivers:sha-$git_sha
```

La progression s'affiche layer par layer. Docker ne transfère que les layers qui ne sont pas déjà présents dans ACR, ce qui rend les pushes suivants beaucoup plus rapides.

### Vérifier que l'image est bien présente dans ACR

```powershell
# Lister les repositories
Get-AzContainerRegistryRepository -RegistryName $acr_name

# Lister les tags disponibles pour notre image
Get-AzContainerRegistryTag -RegistryName $acr_name -RepositoryName "westbalidrivers"
```

```
Registry                      ImageName       Tags
--------                      ---------       ----
acrwestbalidrivers.azurecr.io westbalidrivers {sha-a69596f, staging}
```

### Valider en tirant l'image depuis ACR

Pour s'assurer que l'image est non seulement présente mais récupérable correctement :

```bash
# Supprimer l'image locale pour forcer le pull depuis ACR
docker rmi acrwestbalidrivers.azurecr.io/westbalidrivers:staging

# Pull depuis ACR et lancer
docker run --rm -p 8080:80 acrwestbalidrivers.azurecr.io/westbalidrivers:staging
```
```
PS /home/louis/app> docker run --rm -p 8080:80 acrwestbalidrivers.azurecr.io/westbalidrivers:staging
Unable to find image 'acrwestbalidrivers.azurecr.io/westbalidrivers:staging' locally
staging: Pulling from westbalidrivers
Digest: sha256:573ed224d59d90a552b386eb45d4b30171fe14131a06cde6b515bd69c6233958
Status: Downloaded newer image for acrwestbalidrivers.azurecr.io/westbalidrivers:staging
```

Le site doit répondre sur `http://localhost:8080`, exactement comme lors du test local au chapitre précédent, mais cette fois l'image provient du registre Azure.

<br>

## Politique de rétention (optionnel mais recommandé)

Sans politique de rétention, chaque build CI pousse une nouvelle image dans ACR. Au bout de quelques semaines et selon la fréquence des push, les images s'accumulent, et les inutilisées consomment du stockage et donc du budget.

***A finir de rédiger***

---

## Checklist avant de passer à la suite

- [x] L'ACR est créé avec `--admin-enabled false`
- [x] `Connect-AzContainerRegistry` fonctionne sans erreur
- [x] L'image a été construite avec `--build-arg NEXT_PUBLIC_MAPBOX_TOKEN=...`
- [x] L'image `westbalidrivers:staging` est visible dans ACR via `Get-AzContainerRegistryTag`
- [x] Un `docker run` depuis l'image ACR répond correctement sur `localhost:8080`, carte Mapbox comprise
- [ ] La politique de rétention est configurée

---

## Suite

Notre image est maintenant dans un registre privé Azure, accessible depuis notre tenant.  
Dans le prochain chapitre, nous allons l'utiliser pour créer notre **environnement de staging sur Azure Container Apps**. Celui qui simule des conditions réelles avant que le site ne parte, sans container cette fois, vers Azure Static Web Apps en production.