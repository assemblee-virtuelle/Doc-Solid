Authentification
===

**Vous pouvez contribuer directement à cette documentation [ici](https://hackmd.io/xA03xNjdQfOqAzLXbghPlQ?both)**

**Notes** : `solid-auth-client` n'est pas indispensable pour s'authentifier sur Solid, le `OIDCWebClient` permet de faire sensiblement les même choses (à savoir un Login OIDC et une méthode pour savoir si l'on est connectée ou non, renvoyant une session)
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

## Connexion

Avant de faire quoi que ce soit d'intéressant (comme créer des dossiers, afficher ou créer des ressources, etc), vous devez vous identifier à votre serveur Solid.

### ForceUser (Seulement pour test)

Vous avez le choix de vous identifier de force (contourne le système d'authentification)
Pour cela, éditez le fichier `config.json` crée pendant l'initialisation du serveur et rajoutez la ligne suivante : 

```
"forceUser":"https://alice.localhost:8443/profile/card#me"
```

### Auth en utilisant le client `solid-auth-client`

Premièrement, installez la librairie [`solid-auth-client`](https://github.com/solid/solid-auth-client) en utilisant `npm`: 

```
npm install solid-auth-client
```

Ensuite, dans votre code HTML, insérez la ligne suivante 

```HTML
<script src="./node_modules/solid-auth-client/dist/solid-auth-client.bundle.js"></script>
```

OU

```HTML
<script src="https://solid.github.io/solid-auth-client/dist/solid-auth-client.bundle.js"></script>
```

Note: **Le fichier solid-auth-client.bundle.js peut être utilisé seul**

* Dans le navigateur, la librairie est accessible par `solid.auth`
* Sur Node, vous devez lancer un `npm install solid-auth-client` et la librairie sera accessible par `const auth = require('solid-auth-client')`

Testons le login : 

Ajoutez ces lignes dans la balise `<script>` du code html de la page de login : 

```HTML
...
<script>
solid.auth.trackSession(session => {
  if (!session)
    console.log('The user is not logged in')
  else
    console.log(`The user is ${session.webId}`)
})
</script>
```

Rafraichissez la page et regardez s'il n'y a pas d'erreurs dans la console. 

Deux choses sont nécéssaires pour utiliser l'authentification OIDC : 
1. Un appel a `auth.login(issuerUrl)` Une fois que vous connaissez l'URL de votre issuer/identity provider (fournisseur d'identité). 
Savoir sur quel fournisseur l'utilisateur veut s'identifier est la chose la plus ardue de l'authentification décentralisée traditionnelle, et nécéssite soit une fenêtre de popup ou une rangée de boutons (Facebook, Google, Twitter, etc, etc) pour aider à la sélection du fournisseur (Ceci est connu sous le nom du "Nascar problem").
Pour la suite de ce tutorial, l'`issuerUrl` sera `https://localhost:8443`

2. Un appel a `auth.trackSession()` dès que la page est chargée (soit avec l'event DOMContentLoaded ou autre).
Ceci fait deux choses - Ca détecte si un utilisateur s'est identifié (et a sa session sauvegardée dans le `localStore`), et aussi traite la redirection post-login

Ajoutons l'appel à `auth.trackSession()` et voyons à quoi ressemble une session non authentifiée : 

```javascript
const auth = require('solid-auth-client');
const { fetch } = auth;
// auth is now a client instance

// Trigger a trackSession()
auth.trackSession()
  .then(session => {
    if (!session){
    console.log("Not logged in");
    } else {
        this.webid = session.webId;
        this.fetch = auth.fetch;
        //Parse the POD uris here
    }
  })
```

Dans le `else`, pour parser les URI du POD, il faut faire une requête `GET` sur son WebID, et récuperer
l'URI correspondant aux Object suivant (dans un couple de triplets Subject Predicate Object)

```turtle
<https://savincen.localhost:8443/profile/card#me>
    solid:account </> ;  # link to the account uri
    pim:storage </> ;    # root storage
```

Maintenant passons a l'authentification (si la session est vide) : 

```javascript
auth.trackSession()
  .then(session => {
    if (!session){
    console.log("Not logged in");
    } else {
        this.webid = session.webId;
        this.fetch = auth.fetch;
        //Parse the POD uris here
        auth.login(/*account uri*/);
    }
  })
```

Le login vous envoie vers une page HTML du serveur Solid, où votre username et mot de passe seront demandés

Après le login, vous serez redirigés vers votre page originale, et vous devriez avoir un objet Session avec vos credentials.

Mais surtout, la variable fetch passée ci dessous :

```javascript
const auth = require('solid-auth-client');
const { fetch } = auth;
```

envoie maintenant vos informations d'authentification (Id Token etc) au serveur solid cible

Une fois authentifié, testez la fonction `fetch` dans la console du navigateur. 
Par exemple, vous pouvez fetch le fichier `.acl` à la racine du POD d'alice : 

```
fetch('https://alice.localhost:8443/.acl', {})
  .then(response => response.text())
  .then(contents => { console.log(contents) })
```

**Note :** Ceci marche mieux sur firefox, parce que Chrome bloque les appels si le certificat alice.localhost n'est pas ajouté a la liste des exceptions, et vous aurez une erreur `net::ERR_BLOCKED_BY_CLIENT`