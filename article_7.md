# Mettre en place un pipeline CI/CD sur Azure, partie 7 : automatiser la mise en production

<br>

## Où on en est

L'article 6 a mis la production en ligne à la main : on a identifié l'image `sha-<hash>` validée en staging, on l'a marquée `v0.1.0` côté Git et côté ACR, on en a extrait les fichiers statiques, puis on les a publiés sur Static Web Apps avec la SWA CLI. Une bonne dizaine de commandes, deux shells, et un token de déploiement manipulé à la main.

Dans cet article, nous allons transformer cette séquence en un workflow GitHub Actions. L'objectif n'est pas de retirer l'humain de la mise en production, mais de lui retirer les gestes mécaniques pour ne lui laisser que la décision. Concrètement, une mise en production se résumera à deux actions :

1. poser un tag Git annoté sur le commit à promouvoir
2. lancer le workflow avec ce tag en paramètre

Tout le reste (vérifications, tag ACR, extraction, déploiement, contrôle final) sera géré automatiquement.

<br>

## Étape 1 : une identité dédiée à la production

On pourrait réutiliser le service principal du staging, en lui ajoutant une seconde federated credential et quelques rôles. Je préfère une identité distincte, pour une raison simple : si l'identité du staging est un jour compromise ou mal configurée, la production ne doit pas tomber avec elle. Deux pipelines, deux identités, deux périmètres de droits qui ne se recoupent que sur le strict nécessaire.

### Variables de travail

```powershell
$rg       = "rg-westbalidrivers"
$acr_name = "acrwestbalidrivers"
$swa_name = "swa-westbalidrivers-prod"
$app_name = "gh-actions-westbalidrivers-prod"
```

### App registration et service principal

```powershell
$ghProdApp = New-AzADApplication -DisplayName $app_name
$ghProdSp  = New-AzADServicePrincipal -ApplicationId $ghProdApp.AppId

# Client ID  (servira dans les secrets GitHub)
$ghProdApp.AppId
```

### La federated credential, restreinte à `main`

Même mécanique qu'à l'article 5, avec le format de subject immuable imposé par GitHub depuis juillet 2026. On récupère les identifiants numériques du dépôt et de son propriétaire via l'API GitHub, toujours avec un PAT limité à la permission « Metadata : Read-only » sur ce seul dépôt :

```powershell
$headers = @{
    Authorization = "Bearer $GITHUB_TOKEN"
    Accept        = "application/vnd.github+json"
}

$repo = Invoke-RestMethod -Uri "https://api.github.com/repos/lolecat/WestBaliDrivers" -Headers $headers

New-AzADAppFederatedCredential `
  -ApplicationObjectId $ghProdApp.Id `
  -Name "github-main-branch" `
  -Issuer "https://token.actions.githubusercontent.com" `
  -Subject "repo:lolecat@$($repo.owner.id)/WestBaliDrivers@$($repo.id):ref:refs/heads/main" `
  -Audience "api://AzureADTokenExchange"
```

Pour un workflow déclenché par `workflow_dispatch`, le subject du jeton OIDC reflète la **branche depuis laquelle le run a été lancé**. Lancé depuis `main`, il matche. Lancé depuis `staging` ou depuis une branche de travail, l'échange de jeton échoue avec l'erreur `AADSTS700213` déjà rencontrée à l'article 5.

<br>

## Étape 2 : les RBAC

Le workflow de production a besoin de faire exactement trois choses côté Azure :

1. lire et pousser des tags dans ACR
2. récupérer le token de déploiement de la Static Web App
3. lire les propriétés de la Static Web App (son nom d'hôte, pour le contrôle final)

### Côté ACR : "AcrPush"

Le rôle prédéfini `AcrPush` est nécessaire pour poser le tag **"vx.x.x"**, et il inclut le pull. Pas besoin de plus : ce service principal ne pourra ni supprimer d'image, ni modifier la configuration du registre.

```powershell
$acr_id = (Get-AzContainerRegistry -ResourceGroupName $rg -Name $acr_name).Id

New-AzRoleAssignment `
  -ObjectId $ghProdSp.Id `
  -RoleDefinitionName "AcrPush" `
  -Scope $acr_id
```


### Côté Static Web App : un rôle custom

C'est ici que le principe de moindre privilège demande un peu plus d'effort. Le token de déploiement s'obtient via l'action `Microsoft.Web/staticSites/listSecrets/action`. Le plus petit rôle intégré qui la contient est `Website Contributor`, qui ouvre en réalité tout `Microsoft.Web/*` sur son périmètre : modifier la configuration, supprimer la ressource, et bien d'autres choses dont notre pipeline n'a aucun besoin.

On crée donc un rôle sur mesure, limité à deux actions. La méthode ~~[documentée ici](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles-powershell#create-a-custom-role-with-the-psroledefinition-object) consiste à partir d'un rôle existant, à le vider, puis à le remplir~~ (en fait non, doc pas à jour, on commence à avoir l'habitude).  
On va donc faire ça d'une manière sûre, en passant directement par un PUT sur la ressource ARM :

```powershell
$subscription_id = (Get-AzContext).Subscription.Id

# Créer le GUID du rôle en se basant sur le hash de son nom + abonnement
$md5     = [System.Security.Cryptography.MD5]::Create()
$role_id = [guid]::new($md5.ComputeHash(
    [Text.Encoding]::UTF8.GetBytes("$subscription_id/swa-deployment-token-reader")))

$payload = @"
{
  "properties": {
    "roleName": "Static Web App Deployment Token Reader",
    "description": "Lit une Static Web App et récupère son token de déploiement",
    "type": "CustomRole",
    "permissions": [{
      "actions": [
        "Microsoft.Web/staticSites/read",
        "Microsoft.Web/staticSites/listSecrets/action"
      ],
      "notActions": [], "dataActions": [], "notDataActions": []
    }],
    "assignableScopes": [ "/subscriptions/$subscription_id/resourceGroups/$rg" ]
  }
}
"@

$response = Invoke-AzRestMethod -Method PUT `
  -Path "/subscriptions/$subscription_id/providers/Microsoft.Authorization/roleDefinitions/$($role_id)?api-version=2022-04-01" `
  -Payload $payload

if ($response.StatusCode -notin @(200, 201)) {
  throw "Échec ($($response.StatusCode)) : $($response.Content)"
}

$response.StatusCode
```

Quelques précisions :

- `staticSites/read` n'est pas indispensable pour récupérer le token, mais il permet au workflow de lire le nom d'hôte de la Static Web App pour son contrôle final, plutôt que de le coder en dur.
- `AssignableScopes` limite les endroits où ce rôle **peut être** (et non "est") attribué. Le scoper au resource group évite qu'il serve un jour ailleurs dans l'abonnement par inadvertance. 

On l'attribue ensuite sur la Static Web App :

```powershell
$swa_id = (Get-AzStaticWebApp -ResourceGroupName $rg -Name $swa_name).Id

New-AzRoleAssignment `
  -ObjectId $ghProdSp.Id `
  -RoleDefinitionName "Static Web App Deployment Token Reader" `
  -Scope $swa_id
```

> Un rôle custom fraîchement créé peut mettre quelques minutes à se propager. Si `New-AzRoleAssignment` répond que le rôle est introuvable juste après sa création, il suffit de patienter un peu et de relancer.

### Récapitulatif des identités du projet

| Identité | Utilisée par | Droits |
|---|---|---|
| `id-westbalidrivers-staging` (managée) | La Container App, au runtime | `AcrPull` sur l'ACR |
| `gh-actions-westbalidrivers` | Le workflow de staging | `AcrPush` sur l'ACR, `Container Apps Contributor` sur la Container App |
| `gh-actions-westbalidrivers-prod` | Le workflow de production | `AcrPush` sur l'ACR, rôle custom sur la Static Web App |


<br>

## Étape 3 : le secret GitHub

Un seul nouveau secret de dépôt, dans **Settings → Secrets and variables → Actions** :

| Nom | Valeur |
|---|---|
| AZURE_PROD_CLIENT_ID | $ghProdApp.AppId |

`AZURE_TENANT_ID` et `AZURE_SUBSCRIPTION_ID` existent déjà depuis l'article 5 et sont réutilisés tels quels. Comme pour le staging, ce « secret » n'en est pas vraiment un : c'est un identifiant public. Son seul intérêt dans les secrets GitHub est de ne pas le coder en dur dans le YAML.

Et c'est tout. Pas de token Mapbox (on ne construit rien), pas de token Static Web Apps (on le récupère au runtime).

<br>

## Étape 4 : le workflow de production

```yaml
# .github/workflows/production.yml

name: Promote to production

on:
  workflow_dispatch:
    inputs:
      version:
        description: "Tag Git à promouvoir (ex : v0.2.0)"
        required: true
        type: string

permissions:
  id-token: write   # échange OIDC avec Entra ID
  contents: read    # lecture seule : le tag Git est posé par un humain

env:
  VERSION: ${{ inputs.version }}
  IMAGE: acrwestbalidrivers.azurecr.io/westbalidrivers
  RESOURCE_GROUP: rg-westbalidrivers
  SWA_NAME: swa-westbalidrivers-prod
  SWA_CLI_VERSION: "2.0.10"   # version figée, jamais "latest"

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0   # nécessaire pour récupérer les tags

      - name: Resolve tag to commit
        run: |
          commit=$(git rev-parse "refs/tags/${VERSION}^{commit}")
          echo "SHORT_SHA=${commit::7}" >> "$GITHUB_ENV"

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_PROD_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          enable-AzPSSession: true

      - name: Authenticate Docker against ACR
        uses: azure/powershell@v2
        with:
          inlineScript: Connect-AzContainerRegistry -Name "acrwestbalidrivers"
          azPSVersion: "latest"

      - name: Tag image for production
        run: |
          if docker manifest inspect "${IMAGE}:${VERSION}" > /dev/null 2>&1; then
            echo "${VERSION} existe déjà dans ACR : redéploiement ou rollback, aucun retag."
          else
            # Échoue si l'image sha-* n'existe pas : le commit n'est jamais passé par le staging
            docker pull "${IMAGE}:sha-${SHORT_SHA}"
            docker tag  "${IMAGE}:sha-${SHORT_SHA}" "${IMAGE}:${VERSION}"
            docker push "${IMAGE}:${VERSION}"
          fi

      - name: Extract static files
        run: |
          docker create --name swa-extract "${IMAGE}:${VERSION}"
          docker cp swa-extract:/usr/share/nginx/html ./release
          docker rm swa-extract
          cp deploy/swa/staticwebapp.config.json ./release/

      - name: Deploy to Static Web Apps
        uses: azure/powershell@v2
        with:
          inlineScript: |
            $secret = Get-AzStaticWebAppSecret -ResourceGroupName $env:RESOURCE_GROUP -Name $env:SWA_NAME
            $env:SWA_CLI_DEPLOYMENT_TOKEN = ($secret.Property | ConvertFrom-Json).apiKey
            Write-Output "::add-mask::$($env:SWA_CLI_DEPLOYMENT_TOKEN)"

            npx -y "@azure/static-web-apps-cli@$($env:SWA_CLI_VERSION)" deploy "$env:GITHUB_WORKSPACE/release" --env production
            if ($LASTEXITCODE -ne 0) { throw "Échec du déploiement sur Static Web Apps." }

            $hostname = (Get-AzStaticWebApp -ResourceGroupName $env:RESOURCE_GROUP -Name $env:SWA_NAME).DefaultHostname
            "SWA_HOSTNAME=$hostname" | Out-File -FilePath $env:GITHUB_ENV -Append -Encoding utf8
          azPSVersion: "latest"

      - name: Smoke test
        run: curl -sf --retry 5 --retry-delay 5 --retry-all-errors "https://${SWA_HOSTNAME}/" > /dev/null
```

Le workflow reprend, dans l'ordre, la séquence manuelle de l'article 6. Plusieurs choix méritent quelques mots.

### "workflow_dispatch" et le paramètre version

Le workflow ne se déclenche jamais tout seul : il faut le lancer depuis l'onglet **Actions** du dépôt (bouton *Run workflow*), en choisissant la branche `main` et en saisissant la version. C'est la porte d'approbation humaine décrite plus haut.

Pourquoi ne pas plutôt déclencher le workflow sur le push d'un tag `v*`, ce qui ferait économiser un clic ? Parce que le subject OIDC deviendrait alors `ref:refs/tags/v0.2.0`, différent à chaque version. Or une federated credential compare son subject de façon strictement littérale, sans joker : il faudrait en créer une nouvelle à chaque mise en production. Le déclenchement manuel depuis `main` garde un subject stable.

### "${commit::7}" plutôt que "git rev-parse --short"

Le workflow de staging tague ses images avec `${GITHUB_SHA::7}`, soit toujours exactement 7 caractères. `git rev-parse --short`, lui, peut en renvoyer davantage si 7 caractères ne suffisent pas à identifier un commit sans ambiguïté dans le dépôt. Le jour où ça arriverait, le garde-fou chercherait `sha-95ab7cf3` au lieu de `sha-95ab7cf`, et refuserait à tort une image parfaitement valide. Les deux workflows doivent calculer ce hash court exactement de la même manière.


### Le token de déploiement ne quitte jamais son étape

Le token Static Web Apps est récupéré, utilisé et oublié dans **une seule étape** PowerShell. Il est placé dans une variable d'environnement du processus courant, que la SWA CLI lit directement, et n'est jamais écrit dans `$GITHUB_ENV`, ce qui l'aurait exposé à toutes les étapes suivantes. Le `::add-mask::` demande à GitHub de le remplacer par `***` dans les logs, au cas où un outil aurait l'idée de l'afficher.

La SWA CLI est appelée via `npx` avec une **version figée**. Un `npx -y @azure/static-web-apps-cli` sans version téléchargerait la dernière publiée sur npm au moment du run, c'est-à-dire du code tiers non vérifié, exécuté avec un token de déploiement de production en mémoire. C'est la même logique que le `--frozen-lockfile` de l'article 2 : on n'exécute que ce qu'on a choisi d'exécuter.

<br>

## Étape 5 : première promotion automatisée

### Mettre le workflow en place

Un `workflow_dispatch` n'apparaît dans l'interface GitHub que si le fichier de workflow existe sur la **branche par défaut** du dépôt, ici **main**. 

Le fichier `production.yml` suit ensuite le même chemin que n'importe quelle modification : commité sur `staging`, validé par le pipeline de staging, puis fusionné dans `main`.

```bash
git checkout staging
git add -A
git commit -m "Add production workflow"
git push origin staging
```

Une fois le workflow de staging passé au vert et le site de staging vérifié :

```bash
git checkout main
git merge --ff-only staging
git push origin main
```

### Poser le tag de version

Le commit à promouvoir est le dernier commit de `staging`, celui dont l'image vient d'être construite et validée :

```bash
git fetch origin staging
git tag -a v0.2.0 origin/staging -m "test workflow prod"
git push origin v0.2.0
```

Pousser un tag ne déclenche rien : aucun workflow n'écoute les tags. Le tag est posé, mais rien n'est encore déployé.

### Lancer le workflow

> Depuis l'onglet **Actions** : ***Promote to production → Run workflow → branche main → version v0.2.0.***

### Tester le rollback

> Depuis l'onglet **Actions** : ***Promote to production → Run workflow → branche main → version v0.1.0.***

Aucun tag n'est créé, l'artefact est extrait, puis redéployé. Le site de production revient à la version de l'article 6, avec la configuration d'hébergement actuelle.

On relance ensuite le workflow avec `v0.2.0` pour revenir à la dernière version. Un rollback, puis un « roll-forward », sans aucun build : c'est exactement la promesse du *build once, deploy many* posée à l'article 6.

<br>

## Ce que ce chapitre a changé

Une mise en production, c'était à l'article 6 une dizaine de commandes, deux shells et un secret manipulé à la main. C'est désormais :

```bash
git tag -a v0.3.0 origin/staging -m "..."
git push origin v0.3.0
gh workflow run production.yml --ref main -f version=v0.3.0
```

Et derrière ces trois lignes, le workflow vérifie que le commit est bien passé par le staging, refuse de réécrire un tag de version, déploie l'artefact, ne stocke aucun secret et contrôle le résultat. Le rollback est la même commande avec un numéro de version précédent.

Le pipeline complet ressemble maintenant à ceci :

```
Poste local (dev)
        │
        ▼
Branche staging ──push──→ staging.yml
        │                   ├── docker build (runner)
        │                   ├── push ACR : staging + sha-<hash>
        │                   └── nouvelle révision Container App
        │
        ▼
Branche main ──── tag Git annoté vX.Y.Z (à la main)
                    │
                    ▼
              production.yml (workflow_dispatch, à la main)
                    ├── tag ACR vX.Y.Z (même digest, jamais déplacé)
                    ├── extraction des fichiers 
                    ├── ajout du fichier staticwebapp.config.json (depuis main)
                    ├── déploiement Static Web Apps
                    └── smoke test
```

<br>

## Checklist avant de passer à la suite

- [X] L'app registration `gh-actions-westbalidrivers-prod` existe, avec son service principal
- [X] Sa federated credential est scopée à `refs/heads/main`, au format de subject immuable
- [X] Le rôle custom `Static Web App Deployment Token Reader` existe, scopé au resource group, et ne contient que deux actions
- [X] Le service principal de production a `AcrPush` sur l'ACR et le rôle custom sur la Static Web App, et rien d'autre
- [X] Le secret `AZURE_PROD_CLIENT_ID` est renseigné dans GitHub
- [X] `production.yml` est présent sur `main`, avec une version figée de la SWA CLI
- [X] La promotion de `v0.2.0` passe intégralement au vert, et le résumé du run affiche le bon digest
- [X] `v0.2.0` et le `sha-<hash>` correspondant partagent le même digest dans ACR
- [x] Le rollback vers `v0.1.0` puis le retour à `v0.2.0` fonctionnent sans aucun build
- [x] La carte Mapbox fonctionne en production après promotion automatisée

<br>

## Suite

La chaîne est complète : du poste local à la production, chaque étape est soit automatisée, soit une décision humaine explicite et tracée. Il reste un dernier chantier avant les ultimes finitions : le **domaine custom**, ce qui permettra enfin de retirer le `X-Robots-Tag: noindex` provisoire et d'ouvrir le site aux moteurs de recherche. Ce sera l'objet du prochain chapitre.