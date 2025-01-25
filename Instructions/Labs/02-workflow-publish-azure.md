---
lab:
  title: "Labo\_02\_: utiliser GitHub\_Actions pour Azure pour publier une application web sur Azure\_App\_Service"
  module: 'Module 2: Implement GitHub Actions for Azure'
---

# Vue d’ensemble

Dans ce labo, vous découvrirez comment implémenter un workflow GitHub Actions qui déploie une application web sur Azure App Service.

À la fin de ce labo, vous serez en mesure d’effectuer les tâches suivantes :

* Implémenter un flux de travail GitHub Actions pour CI/CD.
* Expliquer les caractéristiques de base des workflows GitHub Actions.

**Durée estimée : 40 minutes**

## Prérequis

* Un **compte Azure** avec un abonnement actif. Si vous n’en avez pas, vous pouvez vous inscrire à une évaluation gratuite dans la page [https://azure.com/free](https://azure.com/free).
    * Un portail web Azure pris en charge [navigateur](https://learn.microsoft.com/azure/azure-portal/azure-portal-supported-browsers-devices).
    * Un compte Microsoft ou un compte Microsoft Entra avec le rôle Contributeur ou Propriétaire dans l'abonnement Azure. Pour plus d’informations, consultez [Répertorier les attributions de rôle Azure à l’aide du portail Azure](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal) et [Afficher et attribuer des rôles d’administrateur dans Azure Active Directory](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal).
* Un compte GitHub. Si vous n’avez pas de compte GitHub utilisable pour ce labo, suivez les instructions disponibles dans [Inscription à un nouveau compte GitHub](https://github.com/join) pour en créer un.

## Instructions

## Exercice 1 : importer eShopOnWeb dans votre référentiel GitHub

Dans cet exercice, vous allez importer le référentiel [eShopOnWeb](https://github.com/MicrosoftLearning/eShopOnWeb) dans votre propre compte GitHub. Le référentiel est organisé de la manière suivante :

| Dossier | Contenu |
| -- | -- |
| **.ado** | Pipelines YAML Azure DevOps |
| **.devcontainer** | Configuration pour le développement à l’aide de conteneurs (localement dans VS Code ou GitHub Codespaces) |
| **infra** | Infrastructure Bicep et ARM en tant que modèles de code utilisés dans certains scénarios de labo |
| **.github** | Définitions de workflows GitHub YAML |
| **src** | Le site web .NET 8 utilisé dans les scénarios de labo |

### Tâche 1 : importer le référentiel eShopOnWeb

1. Dans votre navigateur web, accédez à GitHub [http://github.com](http://github.com) et connectez-vous à l’aide de votre compte.
1. Démarrez le processus d’importation [https://github.com/new/import](https://github.com/new/import).
1. Entrez les informations suivantes dans la page **Importer votre projet dans GitHub**.

    | Setting | Action |
    |--|--|
    | **URL de votre référentiel source** | Entrez `https://github.com/MicrosoftLearning/eShopOnWeb` |
    | **Propriétaire** | Sélectionnez votre alias GitHub |
    | **Nom du référentiel** | Entrez **eShopOnWeb** |
    | **Confidentialité** | Une fois le **Propriétaire** sélectionné, les options de confidentialité s’affichent. Sélectionnez **Public**. |

1. Sélectionnez **Commencer l’importation** et attendez que le processus d’importation se termine.
1. Dans la page du référentiel, sélectionnez **Paramètres**, puis sélectionnez **Actions > Général** dans le volet de navigation de gauche.
1. Dans la section **Autorisations Actions** de la page, sélectionnez l’option **Autoriser toutes les actions et tous les workflows réutilisables**, puis sélectionnez **Enregistrer**.

> **REMARQUE :** eShopOnWeb est un référentiel volumineux et son importation peut prendre 5 à 10 minutes.

## Exercice 2 : créer des ressources Azure et configurer GitHub 

Dans cet exercice, vous créez un principal de service Azure pour autoriser GitHub à accéder à votre abonnement Azure à partir de GitHub Actions. Vous vérifiez et modifiez également le workflow GitHub qui génère, teste et déploie votre site web sur Azure.

### Tâche 1 : créer un principal de service Azure et l’enregistrer en tant que secret GitHub

Dans cette tâche, vous allez créer un groupe de ressources et un principal de service Azure. Le principal de service est utilisé par GitHub pour déployer l’application eShopOnWeb souhaitée.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com).
1. Ouvrez le **Cloud Shell**, puis sélectionnez le mode **Bash**. **Remarque :** vous devez configurer le stockage persistant si c’est la première fois que vous lancez le Cloud Shell.
1. Créez un groupe de ressources avec la commande CLI `az group create` suivante. Remplacez `<location>` par une région près de vous.

    ```
    az group create -n az2006-rg -l <location>
    ```

1. Exécutez la commande suivante pour inscrire le fournisseur de ressources au **Azure App Service** que vous allez déployer ultérieurement dans le labo.

    ```bash
    az provider register --namespace Microsoft.Web
    ```

1. Exécutez la commande suivante pour générer un nom aléatoire pour l’application web que vous déployez sur Azure App Service. Copiez et enregistrez le nom des sorties de commande pour l’utiliser ultérieurement dans ce labo.

    ```
    myAppName=az2006app$RANDOM
    echo $myAppName
    ```

1. Exécutez les commandes suivantes pour récupérer votre ID d’abonnement. Veillez à copier et enregistrer la sortie à partir des commandes, la valeur d’ID d’abonnement est utilisée ultérieurement dans ce labo.

    ```
    subId=$(az account list --query "[?isDefault].id" --output tsv)
    
    echo $subId
    ```

1. Créez un principal de service avec les commandes suivantes. La première commande stocke l’ID du groupe de ressources dans une variable.

    ```
    rgId=$(az group show -n az2006-rg --query "id" -o tsv)

    az ad sp create-for-rbac --name GH-Action-eshoponweb --role contributor --scopes $rgId --json-auth true
    ```

    >**IMPORTANT :** cette commande génère un objet JSON qui contient les identificateurs utilisés pour s’authentifier auprès d’Azure dans le nom d’une identité Microsoft Entra (principal de service). Copiez l’objet JSON pour l’utiliser dans les étapes suivantes. 

1. Dans une fenêtre du navigateur, revenez à votre référentiel GitHub **eShopOnWeb**.
1. Dans la page du référentiel, sélectionnez **Paramètres**, puis sélectionnez **Secrets et variables > Actions** dans le volet de navigation de gauche.
1. Sélectionnez **Nouveau secret de référentiel** et entrez les informations suivantes :
    * **NOM** : `AZURE_CREDENTIALS`
    * **Secret** : entrez l’objet JSON généré lors de la création du principal de service.
1. Sélectionnez **Ajouter un secret**.

### Tâche 2 : modifier et exécuter le workflow GitHub

Dans cette tâche, vous modifiez le workflow GitHub *eshoponweb-cicd.yml* fourni et l’exécutez pour déployer la solution dans votre propre abonnement.

1. Dans une fenêtre de navigateur, revenez à votre référentiel GitHub **eShopOnWeb**.
1. Sélectionnez **<> Code**, puis, dans la branche primaire, sélectionnez le fichier **eshoponweb-cicd.yml** dans le dossier **eShopOnWeb/.github/workflows**. Ce workflow définit le processus CI/CD pour l’application eShopOnWeb.

    ![Capture d’écran montrant l’emplacement du fichier dans la structure de dossiers.](./media/eshop-cid-workflow.png)
1. Sélectionnez **Modifier ce fichier**.
1. Modifiez les champs de la section `env:` du fichier avec les valeurs suivantes.

    | Champ | Action |
    |--|--|
    | RESOURCE-GROUP : | `az2006-rg` |
    | LOCATION : | `eastus` (ou la région que vous avez sélectionnée lors de la création du groupe de ressources.) |
    | TEMPLATE-FILE : | Pas de changements |
    | SUBSCRIPTION-ID : | Votre ID d’abonnement. |
    | WEBAPP-NAME : | Le nom de l’application web généré de manière aléatoire que vous avez créée précédemment dans le labo. |

1. Lisez attentivement la description du workflow, les commentaires sont fournis pour vous aider à comprendre les étapes du workflow.
1. Supprimez les marques de commentaire de la section **on** en haut du fichier en supprimant le `#`. Le workflow se déclenche avec chaque envoi (push) vers la branche primaire et offre également un déclencheur manuel (`workflow_dispatch`).
1. Sélectionnez **Valider les modifications...** en haut à droite de la page.
1. Une fenêtre contextuelle s’affiche. Acceptez les valeurs par défaut (en validant directement sur la branche primaire), puis sélectionnez **Valider les modifications**. Le flux de travail est exécuté automatiquement.

### Tâche 3 : passer en revue l’exécution du workflow GitHub

Dans cette tâche, vous allez passer en revue l’exécution du workflow GitHub et afficher l’application en cours d’exécution.

1.Sélectionnez **Actions** et vous verrez la configuration du workflow avant l’exécution.

1. Sélectionnez la tâche **eShopOnWeb Build and Test** dans la section **All workflows** (Tous les workflows) de la page. 

1. Le workflow se compose de deux opérations : **buildandtest** et **deploy**. Vous pouvez sélectionner l’une ou l’autre opération et afficher sa progression, ou attendre jusqu’à ce que la tâche soit terminée.

1. Accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et au groupe de ressources **az2006-rg** créé précédemment. Notez qu’à l’aide d’un modèle bicep, GitHub Actions a créé un Plan Azure App Service + App Service dans le groupe de ressources. 

1. Sélectionnez la ressource App Service (nom d’application unique généré précédemment), puis sélectionnez **Parcourir** en haut de la page pour afficher l’application web déployée.

## Exercice 3 : nettoyer les ressources

Dans cet exercice, vous supprimez les ressources créées précédemment dans le labo.

1. Accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et démarrez Cloud Shell. Sélectionnez la session d’interpréteur de commandes **Bash**.

1. Exécutez la commande suivante pour supprimer le groupe de ressources `az2006-rg`. Elle supprime également le Plan App Service et l’instance App Service.

    ```
    az group delete -n az2006-rg --no-wait --yes
    ```

    >**Remarque** : la commande s’exécute de façon asynchrone (comme déterminé par le paramètre `--no-wait`). Par conséquent, vous serez en mesure d’exécuter une autre commande Azure CLI immédiatement après au cours de la même session Bash, mais la suppression réelle du groupe de ressources prendra quelques minutes.

## Révision

Dans ce labo, vous avez implémenté un workflow GitHub Actions qui déploie une application web Azure.
