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

## Connexion

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

Testons le login : 

Ajoutez ces lignes dans la balise `<script>` du code html de la page de login : 

```HTML
...
<script>
  // Lib is exported as window.OIDC
  var OIDCWebClient = OIDC.OIDCWebClient
  var options = { solid: true }
  var auth = new OIDCWebClient(options)
  // auth is now a client instance
</script>
```

Rafraichissez la page et regardez s'il n'y a pas d'erreurs dans la console. 

Deux choses sont nécéssaires pour utiliser l'authentification OIDC : 
1. Un appel a `auth.login(issuerUrl)` Une fois que vous connaissez l'URL de votre issuer/identity provider (fournisseur d'identité). 
Savoir sur quel fournisseur l'utilisateur veut s'identifier est la chose la plus ardue de l'authentification décentralisée traditionnelle, et nécéssite soit une fenêtre de popup ou une rangée de boutons (Facebook, Google, Twitter, etc, etc) pour aider à la sélection du fournisseur (Ceci est connu sous le nom du "Nascar problem").
Pour la suite de ce tutorial, l'`issuerUrl` sera `https://localhost:8443`

2. Un appel a `auth.currentSession()` dès que la page est chargée (soit avec l'event DOMContentLoaded ou autre).
Ceci fait deux choses - Ca détecte si un utilisateur s'est identifié (et a sa session sauvegardée dans le `localStore`), et aussi traite la redirection post-login

Ajoutons l'appel à `auth.currentSession()` et voyons à quoi ressemble une session non authentifiée : 

```javascript
  // Lib is exported as window.OIDC
  var OIDCWebClient = OIDC.OIDCWebClient
  var options = { solid: true }
  var auth = new OIDCWebClient(options)
  // auth is now a client instance

  // Using a standard "document loaded" event listener
  //  (equivalent to jQuery's $(document).ready())
  // Trigger a currentSession() check on page load, in case user is logged in already
  document.addEventListener('DOMContentLoaded', () => {
    auth.currentSession()
      .then(session => {
        console.log(session)
      })
  })
```

Dans la console, vous devriez voir quelque chose ressemblant à ça : 

```
Object { 
  credentialType: "access_token", 
  issuer: undefined, 
  authorization: {}, 
  sessionKey: undefined, 
  idClaims: undefined, 
  accessClaims: undefined 
}
```

Maintenant passons a l'authentification (si la session est vide) : 

```javascript
 var OIDCWebClient = OIDC.OIDCWebClient
  var options = { solid: true }
  var auth = new OIDCWebClient(options)
    
  document.addEventListener('DOMContentLoaded', () => {
    auth.currentSession()
      .then(session => {
        console.log('auth.currentSession():', session)

        if (!session.hasCredentials()) {
          console.log('Empty session, redirecting to login')
          
          auth.login('https://localhost:8443')
        } else {
          console.log('hasCredentials() === true')
        } 
      })
  })
```

Le login vous envoie vers une page HTML du serveur Solid, où votre username et mot de passe seront demandés

Après le login, vous serez redirigés vers votre page originale, et vous devriez avoir un objet Session avec vos credentials.

Mais surtout, vous êtes maintenant en possession d'une fonction authentifiée `fetch` ayant vos credentials

Ajoutons la variable globale `fetch` en lui passant la fonction fetch authentifiée : 

```javascript
  var OIDCWebClient = OIDC.OIDCWebClient
  var options = { solid: true }
  var auth = new OIDCWebClient(options)
  var fetch  // set after currentSession()
    
  document.addEventListener('DOMContentLoaded', () => {
    auth.currentSession()
      .then(session => {
        console.log('auth.currentSession():', session)

        fetch = session.fetch
                        
        if (!session.hasCredentials()) {
          console.log('Empty session, redirecting to login')
          
          auth.login('https://localhost:8443')
        } else {
          console.log('hasCredentials() === true')
        } 
      })
  })
```

Une fois authentifié, testez la fonction `fetch` dans la console du navigateur. 
Par exemple, vous pouvez fetch le fichier `.acl` à la racine du POD d'alice : 

```
fetch('https://alice.localhost:8443/.acl', {})
  .then(response => response.text())
  .then(contents => { console.log(contents) })
```

**Note :** Ceci marche mieux sur firefox, parce que Chrome bloque les appels si le certificat alice.localhost n'est pas ajouté a la liste des exceptions, et vous aurez une erreur `net::ERR_BLOCKED_BY_CLIENT`