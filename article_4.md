# Mettre en place un pipeline CI/CD sur Azure, partie 4 : Azure Container Apps

<br>

## Pourquoi Container Apps pour cet environnement

Notre staging a un profil d'usage particulier : testé par intermittence, pas 24h/24, avec des périodes d'inactivité potentiellement longues entre deux sessions de test. C'est précisément le cas d'usage pour lequel **Azure Container Apps** est pensé, car contrairement à Azure Container Instance, que j'avais envisagé initialement, Container Apps sait mettre son container en veille tout seul (scale-to-zero) et facture un tier gratuit généreux, sans qu'on ait à gérer manuellement le cycle de vie du container (créer avant de tester, détruire après).

<br>

## Concepts de base

Deux ressources Azure distinctes entrent en jeu :

- **Le Container Apps Environment** : la coquille réseau et de journalisation qui héberge une ou plusieurs Container Apps. L'analogie la plus proche serait celle d'un cluster Kubernetes (ou plus précisément son VNet sous-jacent) : une frontière réseau réelle, à l'intérieur de laquelle plusieurs apps peuvent cohabiter et se joindre directement par nom.
Et ... ça n'est pas qu'une analogie, car sous le capot, Container Apps tourne en réalité sur un cluster Azure Kubernetes Service géré en coulisses, avec Envoy comme ingress controller, KEDA pour l'autoscaling événementiel (dont le scale-to-zero), et Dapr disponible nativement pour la communication inter-services. Ça explique pourquoi le portail expose directement des concepts K8s bruts : replicas, revisions, probes liveness/readiness/startup, autoscaling, etc.
- **La Container App** : l'unité réellement déployée, construite à partir de notre image ACR. C'est elle qui possède la configuration de scaling, d'ingress, et le lien vers le registre.

Le modèle de facturation retenu ici est le plan **Consumption** : on paie au vCPU-seconde et Gio-seconde réellement consommés, avec un tier gratuit mensuel conséquent, et surtout la capacité à descendre à zéro replica en absence de trafic, donc à zéro coût de calcul pendant ces périodes.

<br>

## Prérequis

- Les resource provider `Microsoft.App` et `Microsoft.ManagedIdentity` doivent être enregistrés sur l'abonnement 
- L'image `acrwestbalidrivers.azurecr.io/westbalidrivers:staging` doit déjà exister dans ACR (article 3)

```powershell
Register-AzResourceProvider -ProviderNamespace Microsoft.ManagedIdentity
Register-AzResourceProvider -ProviderNamespace Microsoft.App
```

<br>

## Etape 1 : Identité managée et permission de pull sur ACR

Avant même de créer l'environnement, il faut résoudre un problème de l'œuf et de la poule : la Container App a besoin d'une identité pour s'authentifier auprès d'ACR au moment de tirer l'image, mais cette identité doit déjà exister et posséder les permissions **avant** que l'app ne tente son premier pull.

Deux types d'identité managée existent sur Azure : 
- *system-assigned* (créée et liée au cycle de vie de la ressource elle-même, donc générée seulement au moment de la création de la Container App, trop tard pour notre besoin)
- *user-assigned* (une ressource Azure à part entière, qu'on peut créer et outiller avant de l'attacher à quoi que ce soit). On utilise la seconde, précisément pour éviter ce problème de séquencement.

### Variables de travail
```powershell
$id_name = "id-westbalidrivers-staging"
$rg       = "rg-westbalidrivers"
$location = "westeurope"
$acr_name = "acrwestbalidrivers" 
```

### Créer l'identité managée, indépendamment de toute ressource qui l'utilisera
```powershell
New-AzUserAssignedIdentity `
  -Name $id_name `
  -ResourceGroupName $rg `
  -location $location

$identity = Get-AzUserAssignedIdentity -Name $id_name -ResourceGroupName $rg
```

Récupérer l'ID de ressource de l'ACR, pour définir précisément le périmètre (`scope`) de la permission accordée :

```powershell
$acr_id = (Get-AzContainerRegistry -ResourceGroupName $rg -Name $acr_name).Id
```

Attribuer le rôle `AcrPull` (droits de lecture seule sur le registre) à cette identité, sur ce périmètre précis :

```powershell
New-AzRoleAssignment `
  -ObjectId $identity.PrincipalId `
  -RoleDefinitionName "AcrPull" `
  -Scope $acr_id
```

C'est le principe de moindre privilège appliqué à la lettre : cette identité ne pourra jamais pousser d'image, ni toucher à quoi que ce soit d'autre dans l'abonnement, elle pourra seulement lire les images de ce registre.

<br>

## Étape 2 : Création de l'environnement Container Apps

```powershell
$cae_name = "cae-westbalidrivers"

New-AzContainerAppManagedEnv `
  -Name $cae_name `
  -ResourceGroupName $rg `
  -location $location
```
Et là ... ça coince : 
```
New-AzContainerAppManagedEnv_CreateExpanded: Invalid request body for environment 'cae-westbalidrivers'. Must contain 'properties'.
```

Un Container Apps Environment a besoin d'une destination de logs configurée. Pas au sens où il faut absolument stocker les logs quelque part, mais au sens où le cmdlet de création a besoin qu'on lui renseigne explicitement ce choix, ne serait-ce que pour dire qu'on ne veut rien stocker.

### Ce que j'attendais, et ce que ça a réellement demandé

Ma première tentative de créer l'environnement avec seulement `-Name`, `-ResourceGroupName` et `-Location` a échoué avec une erreur peu explicite (`Must contain 'properties'`). En creusant, la raison tient à la façon dont ce cmdlet est généré : sans aucun paramètre optionnel renseigné, le corps de la requête envoyée à Azure part avec un objet `properties` vide, qu'Azure rejette purement et simplement.

> Ce genre d'écart entre l'intuition ("un environnement minimal devrait pouvoir se créer avec le strict nécessaire tel que les paramètres obligatoires du cmdlet powershell") et la réalité de l'outillage Azure n'est pas isolé, j'avais déjà râlé là dessus lorsque je préparais l'AZ-104, cf la fin de [cette doc.](https://github.com/lolecat/AZ-104/blob/main/02%20-%20Storage/lab_part6.md#personal-thoughts-what-happened-to-microsofts-engineering-rigor-)  
Les cmdlets du module `Az` sont largement générées automatiquement à partir des spécifications de chaque équipe produit, sans logique métier qui validerait qu'un appel minimal fait sens. Le réflexe à garder : en cas d'erreur peu explicite, vérifier ce que la doc officielle (ou le portail) propose comme options, plutôt que de partir du principe qu'un appel réduit aux paramètres obligatoires suffira.

### La solution la plus simple : ne pas stocker les logs

Le portail Azure propose une option "Don't save logs (Stream logs without storing them)", qui se traduit en PowerShell en renseignant `-AppLogConfigurationDestination` avec une chaîne vide. C'est une valeur explicitement supportée par l'API (au même titre que `log-analytics` ou `azure-monitor`), pas une astuce de contournement. Elle suffit à peupler le `properties` de la requête, sans avoir besoin de provisionner quoi que ce soit d'autre :

```powershell
$cae_name = "cae-westbalidrivers"

New-AzContainerAppManagedEnv `
  -Name $cae_name `
  -ResourceGroupName $rg `
  -location $location `
  -AppLogConfigurationDestination "" 
```

La contrepartie : les logs restent consultables en direct via le flux de logs du container (`Get-AzContainerAppRevision`, ou l'onglet "Log stream" du portail), mais rien n'est stocké : pas d'historique interrogeable après coup. Pour un staging testé ponctuellement, sans besoin d'investiguer un incident survenu la veille, c'est largement suffisant, et ça évite de provisionner une ressource supplémentaire pour un besoin qu'on n'a pas vraiment.

<br>

Récupérer l'ID de cet environnement, nécessaire pour y rattacher notre Container App :

```powershell
$cae_id = (Get-AzContainerAppManagedEnv -Name $cae_name -ResourceGroupName $rg).Id
```

<br>

## Étape 3 : Création de la Container App

La création se fait en assemblant plusieurs objets de configuration en amont, puis en les passant tous à `New-AzContainerApp`. C'est plus verbeux qu'un simple appel avec des paramètres à plat, mais chaque brique reste lisible individuellement.

### La définition du container

```powershell
$container = New-AzContainerAppTemplateObject `
  -Name "westbalidrivers" `
  -Image "acrwestbalidrivers.azurecr.io/westbalidrivers:staging" `
  -ResourceCpu 0.25 `
  -ResourceMemory "0.5Gi"
```

0.25 vCPU et 0.5 Gio sont largement suffisants pour un Nginx servant des fichiers statiques, ce sont d'ailleurs les plus petites tailles disponibles sur le plan Consumption.

### Les identifiants du registre

```powershell
$registryCredential = New-AzContainerAppRegistryCredentialObject -Server "acrwestbalidrivers.azurecr.io" -Identity $identity.Id
```

`-Identity` référence ici l'identité managée créée à l'étape 1 par son ID de ressource complet. Pas de mot de passe, pas de secret à stocker : c'est l'identité elle-même, via le rôle `AcrPull` qu'on lui a attribué, qui autorise le pull.

### La configuration réseau (ingress)

```powershell
$configuration = New-AzContainerAppConfigurationObject `
  -IngressExternal:$true `
  -IngressTargetPort 80 `
  -Registry $registryCredential
```

- `-IngressExternal:$true` : rend l'app accessible depuis Internet (par opposition à un ingress interne, réservé à la communication entre apps du même environnement)
- `-IngressTargetPort 80` : le port interne écouté par notre container Nginx

### Assemblage final

```powershell
New-AzContainerApp `
  -Name "ca-westbalidrivers-staging" `
  -ResourceGroupName $rg `
  -location $location `
  -EnvironmentId $cae_id `
  -Configuration $configuration `
  -TemplateContainer $container `
  -UserAssignedIdentity @($identity.Id) `
  -ScaleMinReplica 0 `
  -ScaleMaxReplica 1
```

> Les cmdlets du module `Az.App` sont d'apparition relativement récente et évoluent encore assez vite. Si une commande renvoie une erreur de paramètre inconnu, un `Get-Help New-AzContainerApp -Full` (ou l'équivalent pour l'objet concerné) donne la signature exacte de la version installée localement.

<br>

## Étape 4 : Le scale-to-zero, concrètement

```powershell
-ScaleMinReplica 0 `
-ScaleMaxReplica 1
```

`ScaleMinReplica 0` est le paramètre qui change tout par rapport à une App Service ou une VM classique : en l'absence de requête entrante pendant un moment, Azure arrête purement et simplement le dernier replica actif. Le coût de calcul retombe à zéro.

La contrepartie, c'est un "cold start" : la première requête qui arrive après une période d'inactivité doit attendre qu'Azure réveille un container (tirer l'image depuis ACR si le cache local a expiré, démarrer Nginx). Pour notre image de quelques dizaines de mégaoctets, Nginx démarre en une fraction de seconde. Ce délai cumulé aux autres reste de l'ordre de quelques secondes, largement acceptable pour un environnement de test.

`ScaleMaxReplica 1` place un plafond bas : pas besoin de scaler horizontalement pour du staging, ce paramètre sert surtout de garde-fou budgétaire en cas de pic de trafic inattendu (un lien partagé un peu trop largement, par exemple).

<br>

> **Optionnel, mais recommandé avant de partager l'URL** : à ce stade, l'app est accessible publiquement par quiconque connaît (ou devine) son URL générée. L'annexe [Restreindre l'accès à Container Apps avec Entra ID](./article_4_annexe.md) couvre la mise en place d'une authentification en amont du container, pour éviter que des scanners ou bots ne déclenchent des réveils du container en continu. Les tests ci-dessous supposent que cette étape n'a pas encore été appliquée (d'où le `200 OK` direct, sans redirection vers une page de connexion).


## Étape 5 : Tests et validation

### Récupérer l'URL générée

```powershell
$app = Get-AzContainerApp -Name "ca-westbalidrivers-staging" -ResourceGroupName $rg
$fqdn = $app.Configuration.IngressFqdn
```

Le résultat ressemble à `ca-westbalidrivers-staging.<hash>.<region>.azurecontainerapps.io`, un nom de domaine généré et géré entièrement par Azure, HTTPS actif par défaut.

### Vérifier que le site répond

```bash
curl -IL https://$fqdn
```

```
HTTP/1.1 301 Moved Permanently
location: https://ca-westbalidrivers-staging.wittyflower-e45a1bcb.westeurope.azurecontainerapps.io/
date: Tue, 04 Aug 2026 08:53:58 GMT
transfer-encoding: chunked

HTTP/2 200 
server: nginx/1.30.4
date: Tue, 04 Aug 2026 08:53:59 GMT
content-type: text/html
content-length: 89459
last-modified: Sat, 01 Aug 2026 19:25:33 GMT
etag: "6a6e482d-15d73"
accept-ranges: bytes
```

Une réponse `200 OK`, et surtout la présence d'un certificat valide (vérifiable directement dans un navigateur via le cadenas vert, pas d'avertissement) confirment que le TLS est bien pris en charge par Azure sans aucune configuration Nginx de notre part.

Vérifier ensuite que la page `/transfers` fonctionne - carte Mapbox comprise, rappel du chapitre 3 - cela suppose que l'image poussée en staging a bien été construite avec `--build-arg NEXT_PUBLIC_MAPBOX_TOKEN=...`.

### Observer le cold start

Pour constater le scale-to-zero en action : laisser l'app sans trafic une dizaine de minutes, puis renvoyer une requête et observer le délai de réponse (nettement plus long que d'habitude, le temps qu'un replica redémarre), suivi d'un retour à une latence normale sur les requêtes suivantes.

```powershell
# Nombre de replicas actifs à un instant donné, utile pour observer la descente à zéro après une période d'inactivité
Get-AzContainerAppRevision -ContainerAppName "ca-westbalidrivers-staging" -ResourceGroupName $rg | Select-Object Name, Replica
```

```
Name                                Replica
----                                -------
ca-westbalidrivers-staging--p7fsgsn       0
```

Après le scale à 0, une nouvelle requête vers le site web depuis un navigateur prendra une dizaine de secondes entre l'exécution et l'affichage du contenu, le temps que la Container App se lance.

<br>

## Checklist avant de passer à la suite

- [x] L'identité managée `id-westbalidrivers-staging` existe, avec le rôle `AcrPull` attribué sur l'ACR
- [x] L'environnement Container Apps est créé
- [x] La Container App répond en HTTPS sur son URL générée, certificat valide
- [x] La page `/transfers` (Mapbox) fonctionne correctement
- [x] Le comportement de scale-to-zero est observé au moins une fois (cold start constaté après inactivité)

<br>

## Suite

Jusqu'ici, chaque étape du pipeline a été exécutée manuellement, à la main, depuis PowerShell, précisément pour valider chaque brique isolément avant de l'automatiser. Dans le prochain chapitre, on assemble tout ça dans un **pipeline GitHub Actions** : build de l'image sur push vers `staging`, push vers ACR, et déploiement automatique sur cette Container App, sans plus aucune intervention manuelle.