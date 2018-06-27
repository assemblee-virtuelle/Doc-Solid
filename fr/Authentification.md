Authentification
===

**Vous pouvez contribuer directement à cette documentation [ici](https://hackmd.io/xA03xNjdQfOqAzLXbghPlQ?both)**



Nous allons toujours utiliser le nom de `alice` pour le nom d'utilisateur.
Cela veut dire que le WebID de l'utilisateur sera `https://alice.nomdedomaine/profile/card#me`

**Attention :** Dans le cadre d'une utilisation de Solid en localhost, vous devrez ajouter dans vos navigateurs une exception de certificat pour `localhost` et `alice.localhost` (et pour tout autre utilisateur)

## Inscription


Sur Solid :
* Allez sur la page d'accueil de votre serveur Solid dans votre navigateur (`https://localhost:8443` en serveur de test). Cliquez sur `Register` et tapez `alice` pour l'username.

Sur un serveur distant : 
* Une requête POST doit être effectuée vers votre serveur Solid au point d'entrée `/api/accounts/new`. Le contenu de la requete doit être en `application/x-www-form-urlencoded` et sous la forme d'une chaîne de caractères : 
    ```
    "username=alice&password=test..."
    ```

 L'inscription crée un dossier de compte d'utilisateur (POD) dans le chemin du dossier fourni dans `root` lors de l'installation.
Celui ci sera nommé `alice.nomdedomaine`.

## Identification

Avant de faire quoi que ce soit d'intéressant (comme créer des dossiers, afficher ou créer des ressources, etc), vous devez vous identifier à votre serveur Solid.

### ForceUser (Seulement pour test)

Vous avez le choix de vous identifier de force (contourne le système d'authentification)
Pour cela, éditez le fichier `config.json` crée pendant l'initialisation du serveur et rajoutez la ligne suivante : 

```
"forceUser":"https://alice.localhost:8443/profile/card#me"
```

### Auth en utilisant le client `oidc-web`

Premièrement, installez la librairie [`oidc-web`](https://github.com/anvilresearch/oidc-web) en utilisant `npm`: 

```
npm install oidc-web
```

Ensuite, dans votre code HTML, insérez la ligne suivante 

```HTML
<script src="./node_modules/oidc-web/dist/oidc-web.min.js"></script>
```

Notes :
* Des tests sont en cours visant a déterminer si il est impératif d'installer la librairie `oidc-web` ou de simplement récuperer le fichier `oidc-web-min.js`.

* Le client web OIDC est exporté comme globale (`window.OIDC`) une fois que la librairie est chargée par le navigateur.