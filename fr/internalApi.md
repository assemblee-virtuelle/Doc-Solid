Documentation Api
===

Pour contribuer directement à la documentation : https://hackmd.io/o2EKUWwTSSqnS7ioopayDw#

Le node-solid-server est bâti sur le micro framework javascript `Express`. 
Le serveur propose une api pour intéragir avec le LDP selon la norme LDP.

L'api du node-solid-server commence par l'attribution de handlers à certaines routes

Par exemple :

L'objet `router` est un objet du framework `Express` paramétrant les routes
```javascript
  router.copy('/*', allow('Write'), copy)
  router.get('/*', index, allow('Read'), header.addPermissions, get)
  router.post('/*', allow('Append'), post)
  router.patch('/*', allow('Append'), patch)
  router.put('/*', allow('Write'), put)
  router.delete('/*', allow('Write'), del)
```


### Fonction Allow

La fonction `allow` est une fonction du node-solid-server. 
Cette fonction retourne une fonction `allowHandler` et fait du travail préliminaire de
gestion de la ressource ciblée. Elle vérifie si la ressource existe, cherche et obtient son ACL, 
et vérifie si l'utilisateur est autorisé a accèder a la ressource.
L'ACL de la ressource cible est stocké dans `req.acl`, c'est une classe `ACLChecker`.
Et la fonction déterminant si l'agent est autorisé est une méthode de la classe `ACLChecker`, renvoyant une 
promesse accomplie si l'utilisateur/l'agent est autorisé, sinon renvoie une erreur HTTP.
Cette méthode vérifie les permissions pour le `mode` passé en paramètre dans la fonction `allow`. 
Si la ressource ciblée est un ACL, la méthode change le `mode` en `Control`
(Par exemple une requête PUT ayant comme paramètre `Write`, sera remplacé en `Control` si la ressource cible est un ACL)
Ensuite il vérifiera les permissions en récupèrant l'ACL de la ressource, et enfin vérifiera les permissions de l'utilisateur
sur la ressource cible selon le `mode` passé. 

## Les handlers de requête HTTP

Le dernier argument de chaque méthode est une fonction `handler`. Les fonctions handler sont spécifique a chaque requête
et se trouvent dans leurs fichiers respectifs (le handler pour la requête HTTP `get` se trouve dans le fichier `/handlers/get.js`)

Dans le cas de la requête `GET`, on peut voir qu'il y a deux handler supplémentaires : `index` et `header.addPermisisons`.

Le handler `index` vérifie si le chemin possède un fichier index.html, et le handler `header.addPermissions` est un 
header décrivant les permissions de l'utilisateur 

Les handler utilisent tous la classe `LDP` gérant les ressources du filesystem. 
La classe `LDP` fournit des méthodes pour chacune des requêtes, les méthodes sont des callback (ou ont une fonction callback en argument, je sais pas comment le dire TODO:)
La classe LDP est passée dans la requête et se trouve dans `req.app.locals.ldp`, `req` étant la variable de requête HTTP

Pour lire les fichiers sur le filesystem, le serveur utilise le package Node `fs`.
Pour traduire les triplets en graph ou inversement, le serveur utilise [`rdflib.js`](https://github.com/linkeddata/rdflib.js)

### HTTP COPY

Le handler `copy` traite les requêtes HTTP COPY important la ressource spécifiée dans le header `Source`
a une destination spécifiée dans le chemin de la requête. Pour le moment, on peux copier seulement des ressources publiques
et principalement prévu dans pour des application de type "Sauvegarder une ressource externe sur Solid"

```
 * Handles HTTP COPY requests to import a given resource (specified in the
 * `Source:` header) to a destination (specified in request path).
 * For the moment, you can copy from public resources only (no auth delegation
 * is implemented), and is mainly intended for use with
 * "Save an external resource to Solid" type apps.
```

### HTTP DELETE

Le handler `del` traite les requêtes HTTP DELETE. 
La route est passée à la methode `del` de la classe LDP, supprimant la ressource si elle existe et renvoyant un status 200

### HTTP PUT

Le handler `put` traite les requêtes HTTP PUT. 
Pareillement, elle est traitée par la méthode `put` de la classe LDP. 
Premièrement, la fonction vérifie si le chemin est un container ou pas, si c'est un container elle renvoie une erreur `409` avec 
le message `PUT not supported on containers, use POST instead`.
Ensuite, si ce n'est pas un container, elle crée le répertoire englobant si nécéssaire.
Enfin, elle écrit sur le fichier cible par un write stream. 
```javascript
const file = stream.pipe(fs.createWriteStream(filePath))
```
`stream` étant la variable requête `req`, et `filePath` l'uri convertie en file path

### HTTP POST

Le handler `post` traite les requêtes HTTP POST. 

Tout d'abord, le handler va vérifier le content-type, si il est de type `application/sparql` ou `application/sparql-update`, 
il va retourner un handler `patch`.
Ensuite, il va vérifier si le container existe, et renvoyer une erreur `405` si la ressource cible n'est pas un container 
(l'uri doit pointer vers un container, et le nom de la ressource est passée dans le header `Slug`)

Si le `Content-type` est `multipart/form-data`, le handler va invoquer la fonction `multi()` gèrant les imports multiples de fichiers 
grace a `Busboy` TODO: Voir comment marche busboy

Sinon, le handler traite la requête en invoquand la méthode `ldp.post` 
Celle ci commence par vérifier si le nom de fichier contient des caractères interdits tels que '|' ou '/'
et renvoie une erreur 500 si c'est le cas.
Ensuite, elle vérifie si le chemin est disponible. Si le chemin est disponible, elle vérifie si c'est un conteneur ou pas, et si 
c'est un conteneur, elle change change la variable `resourcePath` (le chemin vers la ressource initiale) en chemin vers un fichiers `.meta` du container.
Cela signifie que si la ressource est un container, `ldp.put` créera le container et la ressource `.meta` à l'intérieur,
si la ressoruce n'est pas un container, `ldp.put` créera le fichier comme indiqué plus haut.

### HTTP GET

Le handler `get` traite les requêtes HTTP GET. Si la requête est un glob (Voir globbing) il retourne un globHandler, lisant tout les fichiers dans la requête Glob
en regardant si l'agent a les permissions de lecture sur ces ressources, et les concatène en `text/turtle`.
Ce handler utilise `ldp.get`. Cette méthode vérifie si le chemin passé existe dans le filesystem
* Si il existe et que c'est un répertoire :
Renvoie en turtle (fixé pour le container, c'est en TODO sur le serveur) la liste des ressources dans le répertoire
grace a `ldp.listContainer()`, qui lit le répertoire et répète l'opération de lecture pour chaque fichier TODO: a détailler
* Si il existe et que c'est un fichier : 
Crée un stream de lecture et le renvoie en callback au handler `get` parmis d'autres options. 
Si le `content-type` est en `text/html`, il va utiliser le databrowser codé par la team Solid. (L'interface du serveur)
Si le `content-type` est géré par le Solid serveur, il va traduire le fichier pour le `content-type` demandé, sinon il 
renvoie une erreur `406` avec le message `'Cannot serve requested type: ' + contentType`

### HTTP PATCH

Le handler `patch` traite les requêtes HTTP PATCH. 
Le handler commence par extraire les données de la requête, comme le texte, l'uri et le `content-type`.
Il vérifie aussi si la ressource cible existe, lit son graph et vérifie les permissions pour le patch.
Pour lire les données de la ressource cible il utilise `fs.readFile` suivi de `$rdf.parse` de la libraire `rdflib`.
Pour traiter la requête patch, il utilise des parseurs internes a Solid en fonction du `content-type` passé : 
`Content-Type:application/sparql-update` a comme parseur `sparql-update-parser.js`
`Content-Type:text/n3` a comme parseur `n3-patch-parser.js`

Il faut avoir des accès de lecture et d'écriture pour DELETE et des accès de lecture seulement pour WHERE par exemple.
Un DELETE WHERE requiert accès à la lecture et l'écriture
explication : 

```
  // Read access is required for DELETE and WHERE.
  // If we would allows users without read access,
  // they could use DELETE or WHERE to trigger 200 or 409,
  // and thereby guess the existence of certain triples.
  // DELETE additionally requires write access.
```

La requête patch parsée est ensuite ajoutée au graph de la ressource grace a un `$rdf.serialize()` convertissant un graph 
objet en triplets au format texte, et en l'ajoutant directement dans le fichier grâce à un `fs.writeFile`.
Si aucune erreur est survenue, le serveur renvoie un message disant `Patch applied successfully.`
