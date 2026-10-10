# Mettre en place un pipeline CI/CD sur Azure, partie 5 : GitHub Actions pour le staging

<br>

## Où on en est, et ce qu'il manque encore

Les articles 3 et 4 ont posé les deux briques Azure de l'environnement de staging : le registre de containers (ACR) et la Container App qui en tire son image. Chaque étape a été validée à la main, à dessein, avant d'automatiser quoi que ce soit.

Ce chapitre assemble enfin tout ça dans un pipeline **GitHub Actions** : sur chaque push vers la branche `staging`, l'image est buildée, poussée vers ACR, et la Container App est mise à jour vers cette nouvelle image, sans plus aucune intervention manuelle.

On se concentre ici uniquement sur le staging. Le déploiement de la production vers Azure Static Web Apps (branche `main`) suit une logique assez différente, avec ses propres prérequis, cela sera l'objet du chapitre suivant.

Mais avant d'écrire ce workflow de staging, un prérequis manque encore, et il vaut mieux le combler maintenant plutôt qu'en cours de route : **la fédération d'identité entre GitHub et Azure (OIDC)**. Citée comme objectif dès l'article 1 ("Gérer l'authentification entre GitHub Actions et Azure sans secrets statiques"), elle n'a pas encore été configurée. On va donc la mettre en place, puis écrire le workflow qui en dépend.

<br>

## Pourquoi l'authentification OIDC plutôt qu'un secret statique

La méthode "classique" pour connecter un pipeline CI/CD à Azure consiste à créer un "service principal", générer un secret client, et coller ce secret dans les secrets GitHub. Ça fonctionne, mais ce secret est un identifiant statique à longue durée de vie : s'il fuite (log mal filtré, dépôt rendu public par erreur, etc.), quiconque le récupère peut s'authentifier comme ce service principal jusqu'à ce que quelqu'un pense à le révoquer. Et en attendant, il faut aussi penser à le faire tourner régulièrement.

**OpenID Connect (OIDC)**, ici via ce qu'Azure appelle la **workload identity federation** ("fédération d'identité de charge de travail" ... on va rester sur la terminologie originale 🙃), change la donne : au lieu de stocker un secret, on établit une relation de confiance entre Azure et GitHub. À chaque exécution d'un workflow, GitHub émet un jeton d'identité de courte durée (quelques minutes), signé, qui atteste "ce run vient bien du repo X, sur la branche Y". Azure vérifie ce jeton contre la relation de confiance configurée, et si tout correspond, échange ce jeton contre un token d'accès Azure, lui aussi de courte durée.

L'analogie la plus parlante pour moi serait un ticket Kerberos plutôt qu'un mot de passe partagé : rien de statique à stocker, rien à faire tourner, et un jeton qui expire de lui-même en cas de fuite. Aucun secret Azure n'apparaît jamais dans les logs GitHub, ni dans les secrets du dépôt.

<br>

## Étape 1 : créer l'app registration et le service principal pour GitHub Actions

Une app registration seule ne suffit pas pour recevoir des rôles RBAC : il faut aussi le service principal (l'"application d'entreprise") qui lui est adossé. C'est ce service principal, via son `PrincipalId`, qui sera la cible des attributions de rôle.

```powershell
$app_name = "gh-actions-westbalidrivers"

$ghApp = New-AzADApplication -DisplayName $app_name
$ghSp  = New-AzADServicePrincipal -ApplicationId $ghApp.AppId
```

```powershell
# Les 3 identifiants dont GitHub Actions aura besoin pour s'authentifier
$ghApp.AppId                        # Client ID
(Get-AzContext).Tenant.Id           # Tenant ID
(Get-AzContext).Subscription.Id     # Subscription ID
```

Prendre note de ces trois valeurs, elles iront dans les secrets GitHub à l'étape 4.

<br>

## Étape 2 : configurer la fédération d'identité (OIDC)

C'est ici que la relation de confiance avec GitHub est réellement établie. Le paramètre `-Subject` est le plus important : c'est lui qui définit *quel* jeton GitHub sera accepté par Azure.  
Le format `repo:<owner>/<repo>:ref:refs/heads/<branche>` restreint la confiance à un déclenchement précis, une branche précise, du dépôt précis. Cependant, cette syntaxe est obsolète depuis le 15 Juillet 2026. En effet, Github force désormais la génération de subjects "immuables", au format `repo:<owner>@<owner_id>/<repo>@<repo_id>:ref:refs/heads/<branche>`.  

Il nous faut donc requêter l'API Github pour récupérer les IDs nécessaires. Vu que mon dépôt est privé, j'utilise ici un [PAT](https://github.com/settings/personal-access-tokens) (Personal Access Token) pour m'authentifier auprès de l'API Github. La permission "Metada : Read Only" appliquée seulement au repo suffit largement pour cet usage.

```powershell
$headers = @{
    Authorization = "Bearer $GITHUB_TOKEN"
    Accept        = "application/vnd.github+json"
}

$repo = Invoke-RestMethod -Uri "https://api.github.com/repos/lolecat/WestBaliDrivers" -Headers $headers
# De là, on va pourvoir extraire les valeurs $repo.id et $repo.owner.id pour former le champ "subject"

New-AzADAppFederatedCredential `
  -ApplicationObjectId $ghApp.Id `
  -Name "github-staging-branch" `
  -Issuer "https://token.actions.githubusercontent.com" `
  -Subject "repo:lolecat@$($repo.owner.id)/WestBaliDrivers@$($repo.id):ref:refs/heads/staging" `
  -Audience "api://AzureADTokenExchange"
```

**Point d'attention sur `-Subject`** : la comparaison faite par Azure est sensible à la casse et strictement littérale, aucune règle floue ni wildcard sur ce type de credential. Un run déclenché depuis une autre branche, un tag, ou une pull request ne matchera jamais ce Subject, et l'authentification échouera avec une erreur `AADSTS700213: No matching federated identity record found`. C'est le comportement voulu ici : seule la branche staging doit pouvoir s'authentifier.

> D'autres formats de "-Subject" existent pour d'autres cas d'usage (`repo:<owner>/<repo>:environment:<nom>` pour les environnements GitHub, `repo:<owner>/<repo>:pull_request` pour les PR, etc.). On reste ici sur le format le plus simple, un seul `Subject` pour une seule branche, cohérent avec un pipeline où chaque branche a un rôle strictement défini.

<br>

## Étape 3 : attribuer les rôles nécessaires

Même principe de moindre privilège qu'à l'article 4 pour l'identité managée : ce service principal ne doit pouvoir faire que ce que le workflow `staging` fait réellement. Concrètement, deux choses : pousser une image vers ACR, et mettre à jour la Container App.

```powershell
$rg       = "rg-westbalidrivers"
$acr_name = "acrwestbalidrivers"

$acr_id = (Get-AzContainerRegistry -ResourceGroupName $rg -Name $acr_name).Id
$aca_id = (Get-AzContainerApp -Name "ca-westbalidrivers-staging" -ResourceGroupName $rg).Id
```

```powershell
# Autoriser le push des images vers ACR
New-AzRoleAssignment `
  -ObjectId $ghSp.Id `
  -RoleDefinitionName "AcrPush" `
  -Scope $acr_id

# Autorisation de mise à jour de la Container App
New-AzRoleAssignment `
  -ObjectId $ghSp.Id `
  -RoleDefinitionName "Container Apps Contributor" `
  -Scope $aca_id
```

Les deux attributions sont scopées à la ressource précise, pas au resource group entier. Ce service principal ne pourra ni toucher à l'ACR au-delà du push d'images, ni toucher à une autre ressource du groupe.

> Rappel de la répartition des identités du projet : 
> - l'identité *managée* de l'article 4 (`id-westbalidrivers-staging`, rôle `AcrPull`) sert à la Container App pour "**tirer**" (pull) son image au runtime.  
> - Le service principal créé ici sert à GitHub Actions pour **pousser** (push) l'image et piloter la Container App depuis le pipeline. Deux identités distinctes avec deux jeux de permissions distincts, limités au strict nécessaire.

<br>

## Étape 4 : préparer les secrets GitHub

Direction `Settings → Secrets and variables → Actions` du dépôt privé `lolecat/WestBaliDrivers`.
Deux choix s'offrent à nous : **Environment secrets**, ou **Repository secrets**. A ce stade, on va rester  sur **Repository secrets**.

| Nom | Valeur | Rôle |
|---|---|---|
| AZURE_CLIENT_ID | `$ghApp.AppId` | identifie l'app registration pour l'échange OIDC |
| AZURE_TENANT_ID | `(Get-AzContext).Tenant.Id` | le tenant Azure cible |
| AZURE_SUBSCRIPTION_ID | `(Get-AzContext).Subscription.Id` | l'abonnement cible |
| MAPBOX_TOKEN | le token Mapbox du projet | injecté au build de l'image (`--build-arg`) |

Aucun de ces secrets ne transite jamais par un fichier du dépôt, ni par un `.env` committé : c'est tout l'intérêt de les stocker au niveau GitHub plutôt que dans le code. On notera qu'il n'y a ici **aucun secret Azure à proprement parler** : ni mot de passe, ni clé, ni certificat.  
Les trois valeurs `AZURE_*` sont des identifiants publics (des GUID), pas des secrets, c'est là toute la promesse de l'OIDC.
<br>

## Étape 5 : le workflow de staging

`.github/workflows/staging.yml` :

```yaml
name: Deploy to staging

on:
  push:
    branches: [staging]

permissions:
  id-token: write      # nécessaire pour l'échange OIDC
  contents: read

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Compute short SHA
        run: echo "SHORT_SHA=${GITHUB_SHA::7}" >> "$GITHUB_ENV"

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          enable-AzPSSession: true

      - name: Authenticate Docker against ACR
        uses: azure/powershell@v2
        with:
          inlineScript: |
            Connect-AzContainerRegistry -Name "acrwestbalidrivers"
          azPSVersion: "latest"

      - name: Build & push image
        run: |
          docker build \
            -f docker/Dockerfile \
            --target runner \
            --build-arg NEXT_PUBLIC_MAPBOX_TOKEN=${{ secrets.MAPBOX_TOKEN }} \
            -t acrwestbalidrivers.azurecr.io/westbalidrivers:staging \
            -t acrwestbalidrivers.azurecr.io/westbalidrivers:sha-${{ env.SHORT_SHA }} \
            .
          docker push acrwestbalidrivers.azurecr.io/westbalidrivers:staging
          docker push acrwestbalidrivers.azurecr.io/westbalidrivers:sha-${{ env.SHORT_SHA }}

      - name: Deploy new image to Container App
        uses: azure/powershell@v2
        with:
          inlineScript: |
            $container = New-AzContainerAppTemplateObject `
              -Name "westbalidrivers" `
              -Image "acrwestbalidrivers.azurecr.io/westbalidrivers:sha-${{ env.SHORT_SHA }}" `
              -ResourceCpu 0.25 `
              -ResourceMemory "0.5Gi"

            Update-AzContainerApp `
              -Name "ca-westbalidrivers-staging" `
              -ResourceGroupName "rg-westbalidrivers" `
              -TemplateContainer $container
          azPSVersion: "latest"
```

Le workflow reproduit, étape par étape, la séquence manuelle validée aux articles 3 et 4 : authentification, tag `staging` + tag `sha`, push, puis mise à jour de la Container App. Quelques choix méritent d'être expliqués.

- **`permissions: id-token: write`** est ce qui autorise le job à demander un jeton OIDC à GitHub. Sans cette ligne, `azure/login` échoue avec une erreur explicite sur les permissions manquantes, un grand classique du premier run.
- **`Connect-AzContainerRegistry` plutôt que `docker/login-action` ou `az acr login`** : c'est exactement le même cmdlet que celui utilisé manuellement à l'article 3. Puisque `enable-AzPSSession: true` a déjà ouvert une session Az PowerShell authentifiée par OIDC, ce cmdlet configure Docker pour utiliser cette même session, sans mot de passe ni token à manipuler explicitement. Cohérent avec la préférence du projet pour le module `Az` plutôt que l'Azure CLI.
- **Le déploiement pointe vers le tag `sha-...`, pas vers `staging`** : le tag `staging` reste utile comme pointeur "dernière version" pour un `docker pull` manuel, mais la Container App, elle, est explicitement mise à jour vers un digest précis et immuable. C'est le filet de sécurité pour le rollback évoqué dès l'article 3 : retrouver et redéployer une version exacte ne dépend jamais d'un tag mouvant.

> Le build Docker tourne ici sur le runner GitHub (Ubuntu), pas dans WSL2. La commande `docker build` est donc invoquée directement, sans la gymnastique WSL2 propre au poste local. Le contexte de build reste identique à celui de l'article 2, seul l'hôte change.

<br>

## Validation

Une fois le fichier commité sur la branche `staging` :

```bash
git checkout staging
git push origin staging
```

Dans l'onglet **Actions** du dépôt GitHub, le workflow `Deploy to staging` doit se déclencher, et chaque étape doit passer au vert : login OIDC, authentification Docker, build, push, mise à jour de la Container App.

Côté Azure, on confirme qu'une nouvelle révision a bien été créée avec la bonne image (rappel de l'article 4) :

```powershell
Get-AzContainerAppRevision `
  -ContainerAppName "ca-westbalidrivers-staging" `
  -ResourceGroupName "rg-westbalidrivers" | Select-Object Name, Active, CreatedTime, @{Name='Image'; Expression={$_.TemplateContainer.Image}}
```

L'image de la révision active doit porter le tag `sha-...` correspondant au dernier commit poussé sur `staging`. Un dernier `curl -IL` sur l'URL de la Container App (cf. article 4) confirme que le site répond, carte Mapbox comprise, cette fois servi depuis une image construite et déployée entièrement automatiquement.

> Si l'annexe Entra ID de l'article 4 a été appliquée, cette URL redirige désormais vers une page de connexion : la validation `curl` se fait alors sur la redirection, et la vérification visuelle du site nécessite un navigateur authentifié.

<br>

## Checklist avant de passer à la suite

- [X] L'app registration `gh-actions-westbalidrivers` existe, avec un service principal associé
- [X] La fédération d'identité est configurée, scopée à `repo:lolecat@ID/WestBaliDrivers@ID:ref:refs/heads/staging`
- [X] Le service principal a les rôles `AcrPush` (sur l'ACR) et `Container Apps Contributor` (sur la Container App), et rien de plus
- [X] Les 4 secrets sont renseignés dans GitHub (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `MAPBOX_TOKEN`)
- [X] Un push sur `staging` déclenche le workflow et le fait passer intégralement au vert
- [X] Une nouvelle révision de la Container App est active, avec l'image taguée par le SHA du dernier commit
- [X] Le site de staging répond correctement, carte Mapbox comprise

<br>

## Suite

L'environnement de staging est désormais entièrement automatisé : un simple push sur `staging` suffit à builder, publier et déployer la nouvelle version, sans plus toucher à PowerShell. Reste la production. Dans le prochain chapitre, on crée la ressource **Azure Static Web Apps** et on écrit le workflow de la branche `main`, qui suit une logique volontairement différente : pas de container, pas d'ACR, pas d'identité Azure, mais un déploiement direct du dossier `out/` sur Static Web Apps.