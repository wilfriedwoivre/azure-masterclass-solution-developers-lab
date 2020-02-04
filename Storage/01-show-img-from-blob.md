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
```

Gardez de côté les valeurs de **cntName** et **cntNameResize** !

### Ajout d'image HD/4K

Depuis le portail Azure, ajouter des images de hautes définitions dans le premier conteneur.

## APPLICATION

### Création d'une application en .net Core

Créez un dossier de travail et ouvrez le avec Visual Studio Code. Exécutez ensuite l'instruction suivante dans un terminal de l'IDE :

```powershell
dotnet new webapp
```

### Installation du package Azure.Storage.Blobs

Pour pouvoir manipuler les objets blobs du compte de stockage nous allons installer le paquet suivant :

```powershell
dotnet add package Azure.Storage.Blobs
```

### Ajout des informations de connection au compte de stockage

Ajouter le script json suivant dans le fichier *appsettings.json*, pensez à récupérer les valeurs et à les renseigner.

```json
  "StorageInfo":{
    "ConnectionString":"...",
    "ContainerImg":"...",
    "ContainerImgResize":"..."
  }
```

### Récupération des informations de connection au compte de stockage

Pour pouvoir récupérer ces informations dans le code de l'application, implémenter la classe **Storage** correspondant au *setting* ci-dessus !

<details>
  <summary>Spoiler Alert !</summary>
  
```C#
    public class Storage {
        public string ConnectionString { get; set; }
        public string ContainerImg { get; set; }
        public string ContainerImgResize { get; set; }
    }
```

</details>

Ensuite il faut ajouter la classe qui va nous permettre de récupérer les valeurs du fichier *appsettings.json* à l'exécution :

```C#
    using Microsoft.Extensions.Configuration;
    public class AppSettings
    {
        public Storage StorageInfo { get; set; }

        public static AppSettings LoadAppSettings()
        {
            IConfigurationRoot configRoot = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json")
                .Build();
            AppSettings appSettings = configRoot.Get<AppSettings>();
            return appSettings;
        }
    }
```

Attention au *namespace* !

### Implémentation de l'application et test

Pour ce faire, il nous faut implémenter deux classes.  
La première classe à créer, va nous permettre de stocker les informations d'un blob ici une image, il nous faut donc au minimum l'url pour pouvoir l'afficher dans notre application.

<details>
  <summary>Spoiler Alert !</summary>
  
```C#
    public class Img {
        public string Name { get; private set;}
        public string Url { get; private set; }

        public Img(string name, string url) {
            this.Name = name;
            this.Url = url;
        }
    }
```

</details>

Puis il nous faut créer une autre classe avec les méthodes suivantes :

```C#
        public BlobContainerClient CheckStorageContainer(string containerName) {
            // Create a BlobServiceClient object which will be used to create a container client
            BlobServiceClient blobServiceClient = new BlobServiceClient(this._stoCnString);

            var blobcntclient = blobServiceClient.GetBlobContainerClient(containerName);

            if (blobcntclient == null) {
                // Create the container and return a container client object
                blobcntclient = blobServiceClient.CreateBlobContainerAsync(containerName).Result;
            }

            return blobcntclient;
        }
```

```C#
        public List<Img> GetStorageBlobList(BlobContainerClient containerClient){
            List<Img> result = new List<Img>();
            // List all blobs in the container
            foreach (BlobItem blobItem in containerClient.GetBlobs()) {
                result.Add(new Img(blobItem.Name,containerClient.Uri + "/" + blobItem.Name));
            }
            return result;
        }
```

Attention au *namespace* !

<details>
  <summary>Spoiler Alert !</summary>
  
```C#
    public class Common
    {
        private string _stoCnString;

        public Common(string stocnstring){
            this._stoCnString=stocnstring;
        }

        public BlobContainerClient CheckStorageContainer(string containerName) {
            // Create a BlobServiceClient object which will be used to create a container client
            BlobServiceClient blobServiceClient = new BlobServiceClient(this._stoCnString);

            var blobcntclient = blobServiceClient.GetBlobContainerClient(containerName);

            if (blobcntclient == null) {
                // Create the container and return a container client object
                blobcntclient = blobServiceClient.CreateBlobContainerAsync(containerName).Result;
            }

            return blobcntclient;
        }

        public List<Img> GetStorageBlobList(BlobContainerClient containerClient){
            List<Img> result = new List<Img>();
            // List all blobs in the container
            foreach (BlobItem blobItem in containerClient.GetBlobs()) {
                result.Add(new Img(blobItem.Name,containerClient.Uri + "/" + blobItem.Name));
            }
            return result;
        }
    }
```

</details>

Une fois les éléments ci-dessus en place, nous pouvons modifier le fichier *Index.cshtml.cs* afin de récupérer les images depuis le conteneur du compte de stockage Azure :

```C#
        public List<Img> lstImg { get; set; }

        public void OnGet()
        {
            lstImg = this.Run();
        }

        public List<Img> Run() {
            string storageConnectionString = AppSettings.LoadAppSettings().StorageInfo.ConnectionString;
            string cntName = AppSettings.LoadAppSettings().StorageInfo.ContainerImg;

            var cmn = new Common(storageConnectionString);
            BlobContainerClient blobcntclt = cmn.CheckStorageContainer(cntName);
            List<Img> result = cmn.GetStorageBlobList(blobcntclt);

            return result;
        }
```

Et également le fichier *Index.cshtml* pour afficher les images :

```html
<div class="text-center">
    <h3 class="display-4">Demo connexion à un compte de stockage Azure</h3>
    <p>
        @foreach(var imgname in Model.lstImg){<img src="@imgname.Url" alt="@imgname.Name"/><br>}
    </p>
</div>
```

Compilez et exécutez l'application à l'aide des instructions suivantes :

```powershell
dotnet restore

dotnet build

dotnet run
```
