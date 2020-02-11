# AFFICHER LES IMAGES D'UN CONTENEUR AZURE BLOB

## INFRASTRUCTURE

### Création du compte de stockage Azure

Renseignez les trois variables dans les scripts ci-dessous et exécutez les.
Attention aux noms du compte de stockage ainsi que des conteneurs qui doivent être en minuscule.

```bash
rgName=
az group create --name $rgName --location westeurope
```

```bash
stoName=
az storage account create -n $stoName -g $rgName -l westeurope --sku Standard_LRS --kind StorageV2 --access-tier Hot  
cnSTO=$(az storage account show-connection-string -n $stoName -g $rgName -o tsv)  

echo $cnSTO
```

Gardez de côté la valeur de **cnSTO** !

### Création du conteneur

```bash
cntName=
az storage container create -n $cntName --connection-string $cnSTO
cntNameResize=thumbnails
az storage container create -n $cntNameResize --connection-string $cnSTO
```

Gardez de côté les valeurs de **cntName** !

### Ajout d'image HD/4K

Depuis le portail Azure, ajouter des images de hautes définitions dans le premier conteneur.
Il y a quelques exemples sur le site si vous manquez d'idées : [https://unsplash.com/](https://unsplash.com/)

## APPLICATION

### Récupération de l'exemple

Via votre Cloud Shell, commencez par clone le repository suivant : [azure-storage-dotnetcore-lab](https://github.com/wilfriedwoivre/azure-storage-dotnetcore-lab)

```bash
git clone https://github.com/wilfriedwoivre/azure-storage-dotnetcore-lab.git

cd azure-storage-dotnetcore-lab
```

### Installation du package Azure.Storage.Blobs

Pour pouvoir manipuler les objets blobs du compte de stockage nous allons installer le paquet suivant :

```bash
dotnet add package Azure.Storage.Blobs
```

### Ajout des informations de connexion au compte de stockage

Modifier le fichier *appsettings.json*,  avec les paramètres que vous avez récupérer précédemment.

```json
  "StorageInfo":{
    "ConnectionString":"##La valeur contenue dans cnSTO##",
    "ContainerImg":"##La valeur contenue dans cntName##",
    "ContainerImgResize":"thumbnails"
  }
```

### Appel au Blob Storage

Nous allons donc modifier la classe *DataAccess/BlobDataAccess* en y ajoutant les éléments suivants :

Les différents using :

```C#
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Soat.Labs.Models;
```

Les méthodes pour lister les fichiers disponibles dans un Blob Storage

```C#
public BlobContainerClient CheckStorageContainer(string containerName)
{
    // Create a BlobServiceClient object which will be used to create a container client
    BlobServiceClient blobServiceClient = new BlobServiceClient(this.storageConnectionString);

    var blobcntclient = blobServiceClient.GetBlobContainerClient(containerName);

    if (blobcntclient == null)
    {
        // Create the container and return a container client object
        blobcntclient = blobServiceClient.CreateBlobContainerAsync(containerName).Result;
    }

    return blobcntclient;
}

public List<ImageItem> GetStorageBlobList(BlobContainerClient containerClient)
{
    List<ImageItem> result = new List<ImageItem>();
    // List all blobs in the container
    foreach (BlobItem blobItem in containerClient.GetBlobs())
    {
        result.Add(new ImageItem(blobItem.Name, containerClient.Uri + "/" + blobItem.Name));
    }
    return result;
}
```

Une fois les éléments ci-dessus en place, nous pouvons modifier le fichier *Pages/Index.cshtml.cs* afin de récupérer les images depuis le conteneur du compte de stockage Azure :

Tout d'abord les using

```C#
using Azure.Storage.Blobs;
using Soat.Labs.DataAccess;
```

Et enfin compléter la valeur result dans la méthode *Run* comme suit:

```C#
var cmn = new BlobDataAccess(storageConnectionString);
BlobContainerClient blobcntclt = cmn.CheckStorageContainer(cntName);
List<ImageItem> result = cmn.GetStorageBlobList(blobcntclt);
```

### Valider que toutes vos modifications fonctionnenent.

Si vous êtes en local, vous pouvez lancer le site en local via les commandes suivantes :

```bash
dotnet restore
dotnet build
dotnet run
```

Depuis un Cloud Shell on se contentera d'un simple restore/build afin de vérifier qu'on n'a pas fait de fautes de frappes.

```bash
dotnet restore
dotnet build
```

## DEPLOIEMENT

### Création de votre Web App

Pour cela, on va créer une Azure Web App sous Linux avec une version de Node.js en LTS.

Vu que vous avez tous suivi la MasterClass Azure Developer, aucun problème pour vous !

<details>
  <summary>Spoiler Alert !</summary>
  
  Vous pouvez utiliser des commandes az cli afin de créer votre application Web

```bash
az appservice plan create -n planName -g rgName -l westeurope --is-linux --sku B1

az webapp create -n myAppName -p planName -g rgName --runtime "node|lts"
```

</details>


### Déployer votre code via un local git

Durant la masterclass Azure Dev, nous avons fait cette manipulation en utilisant le portail Azure. Cette fois-ci nous allons uniquement utiliser Azure CLI pour cette opération.

Configurons maintenant notre local git sur notre Web App

```bash
az webapp deployment source config-local-git -n myAppName -g rgName
```

Listons les credentials de notre WebApp via la commande

```bash
az webapp deployment list-publishing-credentials -n myAppName -g rgName --query '[publishingUserName,publishingPassword,scmUri]'
```

Il nous reste plus qu'à configurer notre git et push notre code.

```bash
git add .
git commit -m "Update lab"

git remote add azure 'scmUri/myAppName.git'
git push azure master
```

Après quelques secondes (minutes ...), vous pouvez vous rendre sur l'url https://myapp.azurewebsites.net/ et voir les images de votre compte de stockage.
