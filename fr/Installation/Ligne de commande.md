Installation par ligne de commande 
===

**Vous pouvez contribuer directement à cette documentation [ici](https://hackmd.io/oUOJ2NkOTxO5RHJINas3Tw?both)**

Aller a l'installation pour :
- [Serveur de test](#installation-serveur-de-test)
- [Normale](#installation-basique)


Ce tutoriel suppose que votre solid-server tourne en localhost pour le serveur de test, et utilise Let's Encrypt! pour une utilisation non locale. 

Nous allons enregistrer un utilisateur sous le nom `alice`.
Cela veut dire que le WebID de l'utilisateur sera `https://alice.localhost:8443/profile/card#me`

## Prérequis

* Node 8.10
* Installer le `solid-server` npm package : 
    ```
    git clone https://github.com/solid/node-solid-server.git
    cd node-solid-server
    npm install
    ```

## Installation serveur de test

### Prérequis

* Créez un set de certificats auto-signés par les commandes : 

    ```
    openssl genrsa 2048 > ../localhost.key
    openssl req -new -x509 -nodes -sha256 -days 3650 -key ../localhost.key -subj '/CN=*.localhost' > ../localhost.cert
    ```
    **Note :** Dans cet exemple, `localhost.cert` et `localhost.key` sont crées dans un répertoire un niveau plus haut par rapport au répertoire courant, pour que vous ne committez pas accidentellement vos certificats dans `solid` pendant le développement.


### Initialisation

* Initialisez le serveur
    ```
    npm run solid init
    ```
    ou
    ```
    cd bin
    ./solid init
    ```
    Ceci crée un fichier de config.
    
    Ex:
    ```
    {
        "root": "~/LDP", Chemin vers votre racine du LDP
        "port": "8443", port du serveur
        "serverUri": "https://localhost:8443", Uri du serveur
        "webid": true, Si true, utilise l'authentification webid, si false, n'utilise pas d'authentification
        "multiuser": true, Si true, paramètre le serveur pour plusieurs utilisateurs, si false, un utilisateur par serveur
        "mount": "/", Route sur laquelle le ldp sera monté sur le serveur
        "configPath": "~/LDP/.config", Dossier ou la config initiale sera copiée (les templates et vues)
        "configFile":"./config.json", Fichier de configuration du serveur
        "dbPath": "~/LDP/.db", Chemin vers la db stockant les comptes utilisateurs et leur tokens
        "sslKey": "../localhost.key", Clé privée SSL pour https
        "sslCert": "../localhost.cert", certificat pour https
        "corsProxy": "/proxy", route de l'endpoint proxy cors
        "suffixAcl":".acl" Extension des acl
    }
    ```
    
* Démarrez le serveur solid en mode test (Enlève la vérification des certificats SSL, pour pouvoir travailler avec les certificats auto-signés)

    ```
    bin/solid-test start -v
    ```
    ou
    ```
    DEBUG=solid:* bin/solid-test start
    ```

* Créez un compte pour `alice`. Allez sur la page `https://localhost:8443` dans votre navigateur. Cliquez sur `Register` et tapez `alice` pour l'username. Le register crée un dossier de compte d'utilisateur (POD) dans le chemin du dossier fourni dans `root`.
Celui ci sera nommé `alice.localhost`.



## Installation basique
