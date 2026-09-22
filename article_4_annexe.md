# Annexe : Restreindre l'accès à Container Apps avec Entra ID (Easy Auth)

<br>

## Pourquoi cette annexe

L'article 4 déploie la Container App avec un ingress externe, accessible publiquement dès sa création, donc techniquement atteignable par quiconque connaît (ou scanne) son URL générée, même si celle-ci n'est indexée nulle part. Plutôt que de compter sur l'obscurité de cette URL, cette annexe restreint explicitement qui peut l'atteindre, avec authentification Entra ID en amont du container.

<br>

## Le principe (Easy Auth)

Container Apps embarque un mécanisme d'authentification intégré, communément appelé **Easy Auth** : un sidecar géré par Azure qui s'intercale devant le container de l'app, gère le flux OAuth avec Entra ID, valide les tokens, et n'injecte les requêtes vers Nginx qu'une fois l'utilisateur authentifié. Le container lui-même n'a strictement rien à savoir de ce mécanisme : pas de code à écrire, pas de librairie d'auth à intégrer dans le site.

Pour un sysadmin, l'analogie la plus proche est un reverse proxy avec authentification préalable (type `oauth2-proxy` devant un service interne), sauf qu'ici, c'est Azure qui héberge et gère entièrement ce proxy.

<br>

## Étape 1 : Créer l'app registration dans Entra ID

Cette identité applicative est ce qui permet à Entra ID de reconnaître notre Container App comme un client OAuth légitime. Récupérer d'abord l'URL de l'app (nécessaire pour l'URL de callback) :

```powershell
$app = Get-AzContainerApp -Name "ca-westbalidrivers-staging" -ResourceGroupName $rg
$fqdn = $app.Configuration.IngressFqdn

$redirectUri = "https://$fqdn/.auth/login/aad/callback"
```

```powershell
$adApp = New-AzADApplication `
  -DisplayName "westbalidrivers-staging-auth" `
  -Web @{ RedirectUri = @($redirectUri) }

# Générer un secret client pour cette app registration
$clientSecret = New-AzADAppCredential -ObjectId $adApp.Id
```

> Le module `Az.Resources` (qui gère les app registrations Entra ID) a, comme `Az.App`, changé de signature plusieurs fois ces dernières versions. Si `-Web` renvoie une erreur de paramètre, `Get-Help New-AzADApplication -Full` donne la structure exacte attendue par la version installée.

<br>

## Étape 2 : Stocker le secret et configurer l'authentification

Le secret client doit être enregistré comme un secret de la Container App, puis référencé par nom dans la configuration d'auth, jamais en clair dans la configuration elle-même :

```powershell
$secretObject = New-AzContainerAppSecretObject `
  -Name "entra-client-secret" `
  -Value $clientSecret.SecretText

Update-AzContainerApp `
  -Name "ca-westbalidrivers-staging" `
  -ResourceGroupName $rg `
  -Secret $secretObject

$identityProvider = New-AzContainerAppIdentityProviderObject `
  -RegistrationAppId $adApp.AppId `
  -RegistrationAppSecretSettingName "entra-client-secret"

New-AzContainerAppAuthConfig `
  -Name "current" `
  -ContainerAppName "ca-westbalidrivers-staging" `
  -ResourceGroupName $rg `
  -PlatformEnabled `
  -GlobalValidationUnauthenticatedClientAction "RedirectToLoginPage" `
  -IdentityProvider $identityProvider
```

`-GlobalValidationUnauthenticatedClientAction "RedirectToLoginPage"` est le paramètre qui fait tout le travail : toute requête non authentifiée est redirigée vers l'écran de connexion Entra ID **avant** d'atteindre le container Nginx. Un bot ou un scanner qui tombe sur l'URL se heurte à cette page de connexion, jamais au site lui-même.

<br>

## Étape 3 : Restreindre aux comptes du bon tenant

Par défaut, l'app registration Entra ID peut accepter des comptes de n'importe quel tenant Microsoft. Pour une restriction utile, il faut vérifier que la configuration limite bien l'accès à un tenant spécifique : le paramètre `issuer` de la configuration doit être scopé au tenant précis auquel l'on souhaite faire référence (`https://login.microsoftonline.com/<TENANT_ID>/v2.0`), pas au endpoint multi-tenant générique.

<br>

## Étape 4 : Vérification

```bash
curl -IL https://$fqdn
```

Sans être authentifié, cette requête doit maintenant retourner une redirection vers la page de connexion Entra ID plutôt qu'un `200 OK` direct : c'est la preuve que Easy Auth fonctionne. Une fois connecté via un navigateur avec un compte du bon tenant, le site doit s'afficher normalement.

<br>

## Un effet de bord utile !

Cette restriction règle au passage un questionnement légitime quant au comportement du scale-to-zero face aux bots : un bot ou un scanner qui n'a ni la bonne IP ni les bons identifiants ne déclenche jamais de réveil du container, quel que soit son comportement de scan. La redirection vers Entra ID intervient avant même que la requête n'atteigne l'app. De fait, on peut garantir une gestion des coûts "garantie" de notre Azure Container App, sans scale up intempestifs ou indésirés.

<br>

## Checklist

- [ ] L'app registration Entra ID existe, avec le bon `RedirectUri` et un secret généré
- [ ] `New-AzContainerAppAuthConfig` est en place, avec `RedirectToLoginPage` comme action par défaut
- [ ] Le paramètre `issuer` est scopé au tenant, pas au endpoint multi-tenant
- [ ] Une requête anonyme est bien redirigée vers la page de connexion Entra ID
- [ ] La Container App répond normalement une fois authentifié avec un compte du bon tenant

<br>