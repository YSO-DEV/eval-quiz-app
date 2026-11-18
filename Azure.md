# Déploiement Azure – Application Quiz

Ce document regroupe les commandes utilisées pour déployer l'application Quiz sur Azure, ainsi que les vérifications effectuées pendant le déploiement.

L'objectif est de déployer l'application sous forme de conteneurs Docker avec Azure Container Apps.

## 1. Connexion à Azure

```powershell
# Se connecter à Azure
az login

# Lister les abonnements disponibles
az account list --output table

# Choisir l'abonnement Azure for Students
az account set --subscription "Azure for Students"

# Vérifier l'abonnement actif
az account show --output table

# Récupérer l'identifiant du compte connecté
$USER_ID = az ad signed-in-user show --query id -o tsv

# Vérifier les rôles attribués au compte
az role assignment list --assignee $USER_ID --output table
```

On commence par se connecter à Azure et sélectionner le bon abonnement. La vérification des rôles permet de s'assurer que le compte possède les autorisations nécessaires pour créer et gérer les ressources.

Le rôle Contributor permet de gérer les ressources, tandis que le rôle Owner permet également de gérer les autorisations.

## 2. Enregistrer les fournisseurs de ressources

```powershell
# Enregistrer les fournisseurs nécessaires au projet
az provider register --namespace Microsoft.App --wait
az provider register --namespace Microsoft.ContainerRegistry --wait
az provider register --namespace Microsoft.OperationalInsights --wait

# Vérifier les autorisations
az role assignment list --assignee $USER_ID --output table
```

Azure utilise des fournisseurs de ressources pour activer les différents services dans un abonnement.

- Microsoft.App : Azure Container Apps.
- Microsoft.ContainerRegistry : Azure Container Registry.
- Microsoft.OperationalInsights : services liés notamment aux logs et à la supervision.

## 3. Vérifier les régions disponibles

```powershell
# Vérifier les régions autorisées par la stratégie de l'abonnement
az policy assignment show `
  --name "sys.regionrestriction" `
  --query "parameters.listOfAllowedLocations.value" `
  -o table
```

L'abonnement Azure for Students peut imposer des restrictions sur les régions utilisables. On vérifie donc les régions autorisées avant de créer les ressources.

Pour ce projet, la région choisie est France Central (`francecentral`).

## 4. Définir les variables du projet

```powershell
# Région Azure
$LOCATION = "francecentral"

# Groupe de ressources
$RESOURCE_GROUP = "rg-quiz"

# Registre Azure Container Registry
$ACR_NAME = "quizacr755520"

# Environnement Container Apps
$ENV_NAME = "env-quiz"

# Noms des applications
$BACKEND_APP = "quiz-backend"
$NGINX_APP = "quiz-nginx"
```

On définit les variables au début pour éviter de répéter les mêmes valeurs dans toutes les commandes et limiter les erreurs de frappe.

Le nom du registre ACR est unique à l'échelle d'Azure. Pour générer un autre nom lors d'une nouvelle installation, on peut utiliser :

```powershell
"quizacr$(Get-Random -Minimum 100000 -Maximum 999999)"
```

Pour ce déploiement, le nom réellement utilisé est `quizacr755520`.

## 5. Créer le groupe de ressources

```powershell
az group create `
  --name $RESOURCE_GROUP `
  --location $LOCATION
```

Le groupe de ressources est un conteneur logique qui permet de regrouper les ressources Azure d'un même projet.

Il facilite leur organisation et leur gestion.

## 6. Créer Azure Container Registry (ACR)

```powershell
az acr create `
  --resource-group $RESOURCE_GROUP `
  --name $ACR_NAME `
  --sku Basic `
  --admin-enabled false

# Récupérer l'identifiant complet du registre
$ACR_ID = az acr show `
  --name $ACR_NAME `
  --resource-group $RESOURCE_GROUP `
  --query id -o tsv
```

Azure Container Registry est un registre privé permettant de stocker les images Docker.

Il joue un rôle similaire à Docker Hub, mais il est directement intégré à Azure.

L'offre Basic est utilisée pour limiter les coûts. L'option `--admin-enabled false` désactive le compte administrateur classique du registre. L'authentification sera réalisée avec une identité managée.

## 7. Vérifier les quotas disponibles

```powershell
az containerapp list-usages `
  --location $LOCATION `
  --output table
```

Avant de déployer les applications, on vérifie les quotas disponibles dans la région choisie afin de s'assurer que l'abonnement permet de provisionner les ressources nécessaires.

## 8. Créer l'environnement Azure Container Apps

```powershell
az containerapp env create `
  --name $ENV_NAME `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION

# Vérifier la création de l'environnement
az containerapp env show `
  --name $ENV_NAME `
  --resource-group $RESOURCE_GROUP `
  --query "{name:name, provisioningState:properties.provisioningState, location:location}" `
  --output table
```

Un environnement Container Apps est un espace commun dans lequel plusieurs applications conteneurisées peuvent être déployées.

Il permet notamment de gérer leur environnement d'exécution et leur communication réseau.

Nous utilisons le même environnement pour Nginx et le backend.

## 9. Se connecter à ACR et préparer les images Docker

```powershell
# S'authentifier auprès du registre Azure
az acr login --name $ACR_NAME

# Associer les noms du registre Azure aux images locales
docker tag eval-quiz-app-frontend:latest `
  quizacr755520.azurecr.io/eval-quiz-app-frontend:latest

docker tag eval-quiz-app-backend:latest `
  quizacr755520.azurecr.io/eval-quiz-app-backend:latest

docker tag eval-quiz-app-db:latest `
  quizacr755520.azurecr.io/eval-quiz-app-db:latest
```

La commande `docker tag` permet d'associer une nouvelle référence à une image Docker locale.

Le format utilisé est :

`nom-du-registre.azurecr.io/nom-de-l-image:tag`

On prépare les trois images initiales : frontend, backend et base de données.

## 10. Envoyer les images dans ACR

```powershell
# Envoyer les images dans Azure Container Registry
docker push quizacr755520.azurecr.io/eval-quiz-app-frontend:latest
docker push quizacr755520.azurecr.io/eval-quiz-app-backend:latest
docker push quizacr755520.azurecr.io/eval-quiz-app-db:latest

# Vérifier les dépôts présents dans ACR
az acr repository list `
  --name $ACR_NAME `
  --output table
```

La commande `docker push` envoie les images locales vers le registre Azure.

Elles pourront ensuite être téléchargées par Azure Container Apps pour démarrer les conteneurs.

L'image PostgreSQL a également été poussée dans ACR, mais elle n'est pas utilisée pour le déploiement cloud final : la base de données est hébergée sur Neon.

## 11. Configurer l'identité managée

Nous utilisons une identité managée attribuée par l'utilisateur, nommée `id-quiz-acr`.

Elle permet aux Container Apps de s'authentifier auprès d'ACR sans stocker de mot de passe dans les applications.

```powershell
# Récupérer l'identifiant de l'identité managée
$IDENTITY_ID = az identity show `
  --name "id-quiz-acr" `
  --resource-group $RESOURCE_GROUP `
  --query id -o tsv

# Récupérer le principalId
$IDENTITY_PRINCIPAL_ID = az identity show `
  --name "id-quiz-acr" `
  --resource-group $RESOURCE_GROUP `
  --query principalId -o tsv

# Vérifier le mode d'attribution des rôles d'ACR
az acr show `
  --name $ACR_NAME `
  --resource-group $RESOURCE_GROUP `
  --query roleAssignmentMode `
  --output tsv
```

L'identité existe déjà dans notre projet. Ces commandes permettent de récupérer ses identifiants sans avoir à les saisir manuellement.

### Attribuer le rôle AcrPull

```powershell
az role assignment create `
  --assignee-object-id $IDENTITY_PRINCIPAL_ID `
  --assignee-principal-type ServicePrincipal `
  --role AcrPull `
  --scope $ACR_ID
```

Le rôle `AcrPull` autorise l'identité à télécharger les images depuis ACR.

Il s'agit du principe du moindre privilège : l'identité reçoit uniquement les droits nécessaires à cette opération.

## 12. Créer le backend dans Azure Container Apps

```powershell
az containerapp create `
  --name $BACKEND_APP `
  --resource-group $RESOURCE_GROUP `
  --environment $ENV_NAME `
  --image quizacr755520.azurecr.io/eval-quiz-app-backend:latest `
  --target-port 3007 `
  --ingress internal `
  --min-replicas 1 `
  --max-replicas 1 `
  --env-vars NODE_ENV=production PORT=3007 `
  --user-assigned $IDENTITY_ID `
  --registry-server quizacr755520.azurecr.io `
  --registry-identity $IDENTITY_ID
```

Le backend Node.js est déployé dans une Container App dédiée.

Quelques paramètres importants :

- `--image` : image Docker utilisée pour démarrer le backend.
- `--target-port 3007` : port sur lequel le backend écoute dans le conteneur.
- `--ingress internal` : rend le backend accessible uniquement depuis l'environnement Container Apps.
- `--min-replicas 1` : conserve au moins une réplique active.
- `--max-replicas 1` : limite le nombre de répliques à une.
- `NODE_ENV=production` : indique que l'application fonctionne en production.
- `PORT=3007` : définit le port d'écoute du backend.
- `--user-assigned` : associe l'identité managée à la Container App.
- `--registry-server` : indique le registre contenant l'image.
- `--registry-identity` : indique l'identité utilisée pour télécharger l'image.

Le backend n'est pas exposé directement sur Internet. Les utilisateurs doivent passer par Nginx.

## 13. Configurer les secrets du backend

Le backend a besoin de plusieurs informations sensibles :

- La clé API Resend pour l'envoi des e-mails.
- L'URL de connexion PostgreSQL pour accéder à Neon.

Ces informations sont stockées dans les secrets Azure et non directement dans le code.

### Ajouter les secrets

Dans le portail Azure :

1. Ouvrir la Container App `quiz-backend`.
2. Aller dans la section Secrets.
3. Ajouter le secret `resend-api-key` avec la clé API Resend.
4. Ajouter le secret `database-url` avec l'URL de connexion PostgreSQL de Neon.
5. Enregistrer les modifications.

Pour la connexion à Neon, on utilise de préférence l'URL pooled fournie par le service.

Ne pas inscrire les vraies valeurs dans le fichier README ou dans Git.

### Associer les secrets aux variables d'environnement

```powershell
az containerapp update `
  --name $BACKEND_APP `
  --resource-group $RESOURCE_GROUP `
  --set-env-vars `
  RESEND_API_KEY=secretref:resend-api-key `
  DATABASE_URL=secretref:database-url
```

La syntaxe `secretref:` permet de relier une variable d'environnement à un secret Azure.

Le backend peut ensuite récupérer ces informations avec `process.env.RESEND_API_KEY` et `process.env.DATABASE_URL`.

### Vérifier les variables configurées

```powershell
az containerapp show `
  --name $BACKEND_APP `
  --resource-group $RESOURCE_GROUP `
  --query "properties.template.containers[0].env[].{variable:name,secretReference:secretRef}" `
  --output table
```

Cette commande permet de vérifier que les variables d'environnement et leurs références aux secrets sont bien présentes sans afficher les valeurs sensibles.

## 14. Vérifier le backend

```powershell
# Vérifier la configuration générale
az containerapp show `
  --name $BACKEND_APP `
  --resource-group $RESOURCE_GROUP `
  --output json

# Consulter les logs du backend
az containerapp logs show `
  --name $BACKEND_APP `
  --resource-group $RESOURCE_GROUP `
  --type console `
  --tail 40
```

Les logs indiquent que le backend démarre correctement :

`Server running on port 3007`

Cela confirme que le processus Node.js est lancé, mais ne suffit pas à prouver que la connexion à Neon et les routes API fonctionnent.

## 15. Construire l'image Nginx

L'image Nginx est construite après le déploiement du backend.

C'est nécessaire car le Dockerfile récupère les fichiers du frontend depuis ACR et adapte la configuration Nginx avec l'adresse du backend Azure.

Le Dockerfile utilisé se trouve dans :

`database/nginx/Dockerfile`

Il permet de :

- Partir de l'image frontend déjà présente dans ACR.
- Récupérer les fichiers statiques du frontend.
- Utiliser une image Nginx comme image finale.
- Copier les fichiers du frontend dans le répertoire servi par Nginx.
- Copier la configuration Nginx existante dans l'image.
- Adapter l'adresse de l'upstream backend pour le déploiement Azure.

La configuration Nginx d'origine reste inchangée dans le dépôt.

### Construire l'image Nginx

```powershell
docker build `
  -f .\database\nginx\Dockerfile `
  -t quizacr755520.azurecr.io/eval-quiz-app-nginx:backend-fqdn `
  .\database\nginx
```

### Vérifier la configuration intégrée dans l'image

```powershell
docker run --rm `
  --entrypoint sh `
  quizacr755520.azurecr.io/eval-quiz-app-nginx:backend-fqdn `
  -c "grep -n 'server ' /etc/nginx/conf.d/default.conf"
```

Cette commande permet de vérifier quelle adresse du backend est inscrite dans la configuration Nginx de l'image.

### Envoyer l'image Nginx dans ACR

```powershell
docker push quizacr755520.azurecr.io/eval-quiz-app-nginx:backend-fqdn
```

L'image Nginx est poussée après les images frontend et backend, car sa construction dépend de l'image frontend déjà disponible dans ACR et de l'adresse du backend créé dans Azure.

## 16. Créer la Container App Nginx

```powershell
# Nom de l'application Nginx
$NGINX_APP = "quiz-nginx"

# Créer la Container App
az containerapp create `
  --name $NGINX_APP `
  --resource-group $RESOURCE_GROUP `
  --environment $ENV_NAME `
  --image quizacr755520.azurecr.io/eval-quiz-app-nginx:latest `
  --target-port 80 `
  --ingress external `
  --min-replicas 1 `
  --max-replicas 1 `
  --user-assigned $IDENTITY_ID `
  --registry-server quizacr755520.azurecr.io `
  --registry-identity $IDENTITY_ID
```

Nginx est le point d'entrée public de l'application.

- `--target-port 80` : Nginx écoute sur le port 80 dans son conteneur.
- `--ingress external` : autorise l'accès depuis Internet.
- `--user-assigned` : associe l'identité managée.
- `--registry-server` et `--registry-identity` : permettent de télécharger l'image depuis ACR.

### Mettre à jour l'image utilisée par Nginx

Après la création initiale, nous avons mis à jour la Container App pour utiliser la nouvelle image construite avec le tag `backend-fqdn`.

```powershell
az containerapp update `
  --name $NGINX_APP `
  --resource-group $RESOURCE_GROUP `
  --image quizacr755520.azurecr.io/eval-quiz-app-nginx:backend-fqdn
```

## 17. Consulter les logs de Nginx

```powershell
az containerapp logs show `
  --name $NGINX_APP `
  --resource-group $RESOURCE_GROUP `
  --type console `
  --tail 50
```

Les logs permettent de vérifier le démarrage de Nginx et les requêtes reçues.

Nous avons constaté que Nginx démarre correctement et répond à la page d'accueil avec un code HTTP 200.

Cela confirme que le frontend est bien servi par Nginx.

## 18. Récupérer les adresses des applications

```powershell
# Adresse publique de Nginx
az containerapp show `
  --name $NGINX_APP `
  --resource-group $RESOURCE_GROUP `
  --query "properties.configuration.ingress.fqdn" `
  --output tsv

# Configuration de l'ingress backend
az containerapp show `
  --name $BACKEND_APP `
  --resource-group $RESOURCE_GROUP `
  --query "properties.configuration.ingress" `
  --output json
```

L'adresse publique de Nginx permet d'accéder à l'application depuis un navigateur.

Le backend possède un ingress interne : son adresse ne doit pas être utilisée directement depuis le navigateur.

## 19. Vérifier les ressources déployées

```powershell
# Lister les Container Apps du groupe de ressources
az containerapp list `
  --resource-group $RESOURCE_GROUP `
  --output table

# Vérifier l'environnement
az containerapp env show `
  --name $ENV_NAME `
  --resource-group $RESOURCE_GROUP `
  --output table

# Lister les images présentes dans ACR
az acr repository list `
  --name $ACR_NAME `
  --output table
```

Ces commandes permettent d'avoir une vue d'ensemble des ressources créées et des images disponibles.

---

# État du déploiement

## Ce qui fonctionne

- Le groupe de ressources `rg-quiz` est créé.
- Le registre ACR `quizacr755520` est créé.
- L'environnement Container Apps `env-quiz` est créé.
- Les images Docker frontend, backend et base de données ont été poussées dans ACR.
- L'identité managée `id-quiz-acr` est configurée avec le rôle `AcrPull`.
- Le backend `quiz-backend` est créé et démarre sur le port 3007.
- Le backend possède un ingress interne.
- Les secrets Resend et Neon sont référencés dans ses variables d'environnement.
- L'application Nginx `quiz-nginx` est créée et démarre.
- Nginx est accessible publiquement et sert la page d'accueil du frontend.

## Ce qui reste à résoudre

Les appels API ne fonctionnent pas encore correctement.

Une requête envoyée à travers Nginx retourne une erreur `504 Gateway Time-out`.

Le diagnostic porte sur la communication entre Nginx et le backend :

- Vérifier la résolution DNS du backend depuis Nginx.
- Vérifier l'accès réseau au backend depuis le même environnement Container Apps.
- Vérifier le fonctionnement du reverse proxy.
- Valider les requêtes API.
- Valider la connexion effective à Neon.

Le démarrage des conteneurs est confirmé, mais le fonctionnement complet de l'application n'est pas encore validé.
