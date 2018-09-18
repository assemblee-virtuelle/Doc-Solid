Installation & Authentification (Plusieurs serveurs)
===

**Note** : Le solid-auth-client, utilisé avec un serveur solid lancé par Node, nécéssite une configuration supplémentaire que je n'ai pas réussi a trouver.

Pour installer un serveur solid, veuillez voir la [documentation d'installation](https://github.com/assemblee-virtuelle/Doc-Solid#installation-1) du node-solid-server. 
L'installation de plusieurs serveurs n'a pas de points particuliers. 

J'ai lancé les deux serveurs en utilisant les lignes de commande.
Mon fichier de configuration pour les deux serveurs se présente de cette manière : 
```json
{
"root": "~/SolidPlayground/TestsWebComponentSolid/LDP",
"port": "8443",
"serverUri": "https://localhost:8443",
"webid": true,
"mount": "/",
"configPath": "~/SolidPlayground/TestsWebComponentSolid/LDP/.config",
"configFile": "./config.json",
"dbPath": "~/SolidPlayground/TestsWebComponentSolid/LDP/.db",
"sslKey": "../../localhost.key",
"sslCert": "../../localhost.cert",
"multiuser": true,
"corsProxy": "/proxy"
}
```

Note : La configuration du second serveur est similaire à la configuration du premier, seul le port et le dossier Root sont différents. Sur le premier le port est en 8443, le second en 8444, le dossier root du premier est LDP1 et le dossier root du second est LDP2.

Prenons l'exemple de deux utilisateurs, Alice et Bob, bob voulant accèder a une ressource sur le POD d'Alice, qui est sur un serveur Solid différent. 
* Je me register en tant qu'Alice sur le premier serveur, et en tant que Bob sur le second serveur. Alice a comme WebID `https://localhost:8443/profile#me`, et Bob a comme WebID `https://localhost:8444/profile#me`.
* Sur le POD d'Alice, j'ajoute, dans l'ACL de la ressource que Bob veut requêter, le WebID de Bob et les droits correspondants à ce que Bob veut faire de la ressource (read, write etc)
* En tant que Bob, je m'authentifie a mon serveur, cela renverra au client une fonction `fetch` avec credentials.
* J'essaye d'acceder a la ressource cible sur le serveur d'Alice. La requête HTTP a un header `authorization` possèdant un Bearer récupéré pendant l'authentification de Bob, encodé en base 64.

Bearer Token décodé : 

```json
{
  "iss": "904db75f677eb17c37aee50ab6c2e6a8",
  "aud": "https://Bob.localhost:8443",
  "exp": 1536146598,
  "iat": 1536142998,
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6ImNfaHA4UE0zaGtzIn0[...]",
  "token_type": "pop"
}
```

**Note :** Le champ "id_token" est également encodé en Base64, possède toutes les informations de session d'une personne (Entre autres, son WebID, sa clé publique etc...)

* Le serveur d'Alice vérifiera que le WebID de Bob est présent dans l'ACL de la ressource voulue, et si oui, autorise sa requête. 

Note : Selon la spec, le serveur d'Alice, lors de la requête a une ressource, devrait rediriger Bob vers son serveur pour lui demander ses informations de connexion (similaire a "Connexion par Facebook" faisant apparaitre un popup demandant de se logger sur facebook). En revanche en pratique, il n'y a aucune redirection. 
La raison est que la fenêtre "Authorisation" n'a pas été encore implémentée.

