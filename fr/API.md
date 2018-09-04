API
===

**Pour contribuer directement à cette documentation, cliquez [ici](https://hackmd.io/weWkSQBSTI2q7D2d659wpw?both)**

Solid applique la [Linked Data Platform spec](https://www.w3.org/TR/ldp/) et fournit donc une API REST pour des opérations CRUD sur les ressources et conteneurs. 

## Créer

Pour créer de nouvelles ressources (documents ou conteneurs) en utilisant le LDP, le client doit indiquer au préalable le type de la ressource qui va être créée. Le LDP utilise le header `Link`, qui lorsque référencé, obtient des informations additionnelles sur chaque type de ressource. 
Actuellement, l'implémentation LDP ne supporte que les [Basic Containers](https://www.w3.org/TR/ldp/#ldpbc)

Le LDP offre aussi un système pour que les clients puissent nommer leurs nouvelles ressources grace au header appelé `Slug`.

### Création de conteneur (dossier)

Pour créer une nouvelle ressource de type conteneur basique, le header Link doit fixé à la valeur suivante: `Link: <http://www.w3.org/ns/ldp#BasicContainer>; rel="type"`

Par exemple, pour créer un conteneur basique appelé `data` sur `https://example.org/`, le client devra envoyer la requête POST suivante, avec le header `Content-Type` fixé a `text/turtle`:

REQUEST: 

```httpspec
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Link: <http://www.w3.org/ns/ldp#BasicContainer>; rel="type"
Slug: data
```

Les données additionnelles envoyées dans la requête POST seront ajoutés dans un fichier .meta dans le conteneur.

REPONSE:

```httpspec
HTTP/1.1 201 Created
```

### Créer des documents (fichier)

Pour créer une nouvelle ressource document, le header `Link` doit être fixé à la valeur suivante `Link:<http://www.w3.org/ns/ldp#Resource>; rel="type"`

Par exemple, pour créer une ressource appelée `test` sur `https://example.org/data/`, le client devra envoyer la requête POST suivante, avec le header `Content-Type` toujours set to `text/turtle`

Requête: 

```httpspec
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
Slug: test
```

Les données additionnelles envoyées dans la requête POST seront stockées dans la ressource créée. 

REPONSE:

```httpspec
HTTP/1.1 201 Created
```

Plus d'exemples peuvent être trouvés dans le LDP [Primer Document](https://www.w3.org/TR/ldp-primer/).

#### Créer avec le HTTP PUT

Des ressources peuvent être crées en utilisant la requête HTTP PUT. Bien que HTTP PUT est communément utilisé pour écraser des ressources, cette façon est générallement préférée pour créer des ressources non-RDF (utilisant un mime type différent que `text/turtle` et autres formats RDF supportés). (TODO: A reformuler)

REQUÊTE: 

```httpspec
PUT /picture.jpg HTTP/1.1
Host: example.org
Content-Type: image/jpeg
...
```

RESPONSE:

```httpspec
HTTP/1.1 201 Created
```

Une fonctionnalité de Solid qui n'est pas encore dans la spec LDP est la capacité de traiter les ressources *récursivement*. Cette fonctionnalité est très utile quand le client veut avoir un contrôle absolu sur un URI namespace (Par ex migrer d'un POD a un autre). 
Pour clarifier, vous pouvez créer le chemin complet a une ressource (en comprenant les conteneurs intermédiaires) si le chemin n'existait pas précédemment. (Si vous êtes familiers avec les commandes Unix, imaginez ça comme un mkdir -p). Par exemple, une application de calendrier utilise un URI pattern basé sur des dates pour créer de nouveaux événements (yyyy/mm/dd). Au lieu de faire un POST pour le container année, un POST pour le mois et un POST pour le jour, vous pouvez faire ça en une seule requête POST. 
Pour créer un nouvel événement appelé `event1`:

REQUÊTE: 
```httpspec
PUT /2015/05/01/event1 HTTP/1.1
Host: example.org
```

REPONSE:
```httpspec
HTTP/1.1 201 Created
```

Cette requête créera une ressouce appelée `event1`, ainsi que les ressources intermédiaires -- Les Conteneurs pour le mois `05` et pour le jour `01` dans le conteneur parent `/2015/`. 

**IMPORTANT** Utiliser PUT pour créer des conteneurs seulement n'est pas supporté, parce que le comportement de HTTP PUT (écrasement) n'est pas bien défini pour les conteneurs. Vous **DEVEZ** utiliser POST (comme défini sur le LDP) pour créer des conteneurs seuls. 

## Lire

Les ressources peuvent être accédées (lues) en utilisant des requêtes GET.  

**Important:** Un header par défaut `Content-Type: text/turtle` sera utilisé par défaut sur les ressources RDF ou conteneurs n'ayant pas de header `Accept`. (TODO A TESTER)

En conformité avec la spec LDP, le serveur solid retourne une liste du contenu des conteneurs lors de la réception de requêtes pour conteneurs. Pour chaque ressource dans un conteneur, le serveur solid inclut des metadata additionnelles, comme la date de modification de la ressource, la taille de la ressource, et plus important, n'importe quel type RDF spécifié pour la ressource dans sa metadata.
Dans l'exemple ci-dessous, la ressource `<profile>` a un type RDF additionnel `<http://xmlns.com/foaf/0.1/PersonalProfileDocument>`, et la ressource `<workspace/>` est de Type RDF `<http://www.w3.org/ns/pim/space#Workspace>`

De la metadata additionnelle peut être ajoutée, décrivant telle ressource pointant vers un fichier ou répertoire dans le serveur, en utilisant le [Vocabulaire POSIX](http://www.w3.org/ns/posix/stat#)

Un exemple de requête à la racine : 

REQUÊTE: 
```httpspec
GET /
Host: example.org
```

REPONSE: (TODO a changer)

```httpspec
HTTP/1.1 200 OK
```

```turtle
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .

<>
    a <http://www.w3.org/ns/ldp#BasicContainer>, <http://www.w3.org/ns/ldp#Container>, <http://www.w3.org/ns/posix/stat#Directory> ;
    <http://www.w3.org/ns/ldp#contains> <profile>, <data/>, <workspace/> ;
    <http://www.w3.org/ns/posix/stat#mtime> "1436281776" ;
    <http://www.w3.org/ns/posix/stat#size> "4096" .

<profile>
    a <http://xmlns.com/foaf/0.1/PersonalProfileDocument>, <http://www.w3.org/ns/posix/stat#File> ;
    <http://www.w3.org/ns/posix/stat#mtime> "1434583075" ;
    <http://www.w3.org/ns/posix/stat#size> "780" .
```

### Engloblement

Note: Non testé car ne marche pas dans ma version (fix en cours)

Nous avons vu que dans certaines occasions, utiliser les fonctionnalités LDP n'était pas suffisant. Par exemple, pour optimiser certaines applications, nous dvons agreer toutes les ressources RDF d'un conteneur et les retourner dans une seule requête GET. Le node-solid-serveur implémente cette fonctionnalitée appellée "Globbing". De la même façon que le [glob shell UNIX](https://en.wikipedia.org/wiki/Glob_(programming)), faire une requête GET sur une URI terminant par un `*` retournera une vie aggrégée de toutes les ressources correspondant au modèle(pattern) indiqué. 

Par exemple, prenons `/data/res1` et `/data/res2`, deux ressources contenant un triplet chacune définissant leur type : 

Pour res1: 
```turtle
<> a <https://example.org/ns/type#One> .
```

Pour res2: 
```turtle
<> a <https://example.org/ns/type#Two> .
```

Si vous voulez prendre toutes les ressources commencant par res d'un conteneur (ex `/data/res1`, `/data/res2`) en une requête, vous pouvez faire une requête GET sur l'addresse `/data/res*` : 

REQUÊTE:

```httpspec
GET /data/res* HTTP/1.1
Host: example.org
```

REPONSE:

```httpspec
HTTP/1.1 200 OK
```

```turtle
<res1>
    a <https://example.org/ns/type#One> .

<res2>
    a <https://example.org/ns/type#Two> .
```

Une autre alternative serait de demander aux server de retourner *toutes* les ressource d'un conteneur, comprenant aussi les triplets du conteneur. 

REQUÊTE: 

```httpspec
GET /data/* HTTP/1.1
Host: example.org
```

REPONSE:

```httpspec
HTTP/1.1 200 OK
```

```turtle
<>
    a <http://www.w3.org/ns/ldp#BasicContainer> ;
    <http://www.w3.org/ns/ldp#contains> <res1>, <res2> .

<res1>
    a <https://example.org/ns/type#One> .

<res2>
    a <https://example.org/ns/type#Two> .
```

Note: le processus d'aggrégation n'est pas récursif, il ne s'appliquera donc pas aux conteneurs enfants. 


## EDITER 

Les requêtes de type `POST` ne peuvent pas être utilisées pour écrire sur des ressources existantes, en effet leur comportement leur impose que, si le nom de la ressource cible indiqué dans la requête `POST` existe déjà, le serveur SOLID créera une nouvelle ressource au nom donné + préfixe hashé (Si PlanetList.ttl existe déjà, le serveur créera la ressource 2HH4H538eza-PlanetList.ttl).

Une ressource peut donc être éditée de plusieurs façons : 

#### PUT
En utilisant une requête `PUT`. Les requêtes PUT écrasent une ressource, pour en modifier une avec ce type de requête, la ressource doit donc être au préalable récupérée par une requête GET. Le contenu voulu doit être ajouté a l'ancien, et la somme des deux devra être envoyée par requête `PUT`.

#### PATCH

En utilisant la requête de type `PATCH`, et en envoyant une requête SPARQL en contenu à la ressource cible. Si la rssource n'existe pas, elle devra être crée par une requête POST ou PUT

Par exemple, pour changer la valeur d'un prédicat de type `title`, le client devra envoyer une requête SPARQL `DELETE`, suivie par une requête SPARQL `INSERT`. Plusieurs requêtes SPARQL (délimitées par un `;`) peuvent être envoyées dans la même requête HTTP `PATCH`.

Note: Chaque ressource est son propre endpoint SPARQL, donc l'URI cible devra être celle de la ressource, et le Content-Type envoyé devra être de type `application/sparql-update` (TODO mettre une ressource simple pas un container)

REQUÊTE:

```httpspec
PATCH /data/ HTTP/1.1
Host: example.org
Content-Type: application/sparql-update
```

```turtle
DELETE DATA { <> <http://purl.org/dc/terms/title> "Basic container" };
INSERT DATA { <> <http://purl.org/dc/terms/title> "My data container" }
```


REPONSE:

```httpspec
HTTP/1.1 200 OK
```

**Important** : Il n'y a actuellement aucun support pour les *blank nodes* et *RDF lists* dans les patchs SPARQL.

## SUPPRIMER

Pour supprimer une ressource, vous pouvez simplement envoyer une requête HTTP `DELETE` à la ressource cible pour la supprimer. 

Si vous voulez simplement supprimer des triplets, il faudra le faire par un patch SPARQL.

