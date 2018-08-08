WAC ACLs
===

**Vous pouvez contribuer directement à cette documentation [ici](https://hackmd.io/JxLDg2DwSdqNXLHyqoqaZw?both)**

# Résumé

## WAC

WAC (Web Access Control) est un système de controle d'accès cross-domain. Le concept principal devrait être familier pour les développeurs, vu qu'il est similaire aux controles d'accès utilisés dans des systèmes de fichiers. Son but est de vérifier l'accès aux [agents](#description-des-agents) (utilisateurs, groupes etc) pour effectuer divers opérations (lire, écrire, supprimer, etc) sur les ressources. 
WAC est composé de plusieurs éléments clés : 

1. Les ressources sont identifiés par des URIs, et peuvent référer a n'importe quel document web ou ressource. 
2. Il est *assertif* -- Les politiques de controles d'accès vivent dans des document web réguliers, qui peuvent être exportés/sauvegardés facilement, en utilisant le même méchanisme que pour sauvegarder le reste de vos données. (help in trad needed)
3. Les utilisateurs et groupes sont identifiés par des URLs (spécifiquement, par des WebIDs)
4. Il est *cross-domain* -- Tout ses composants, comme les ressources, les WebIDs d'agents, et même les documents contenant la politique de controle d'accès peuvent potentiellement demeurer sur des domaines séparés. En d'autres termes, vous pouvez donner accès a une ressource sur un site à des utilisateurs et groupes hébergés sur un autre site.

## ACLs

Dans un système utilisant les Web Access Control, chaque ressource a son ensemble de déclarations d'Authorisations décrivants : 
1. Qui a accès a cette ressource (qui sont les *[agents](#description-des-agents)* autorisés)
2. Quel type (ou [*modes*](#modes-daccès)) d'accès ils possèdent

Ces autorisations sont soit spécifiés explicitement pour une ressource, ou (le plus souvent) hérités de leur dossier ou conteneur parent. Dans tout les cas, les déclarations d'Autorisations sont placés dans des document WAC séparés appelés Access Control List Resouces (Ou simplement ACLs).

### Format d'un ACL

Les permissions dans une ressource ACL sont stockées dans un format de Linked Data ([Turtle](https://www.w3.org/TR/turtle/) par défaut, mais aussi disponible dans d'autres sérialisations)

*Note : Une compréhension du Linked Data et des concepts RDF aide a comprendre la terminologie utilisée dans ce document*

WAC utilise l'ontologie http://www.w3.org/ns/auth/acl pour ses termes. Dans le reste du document, le préfixe `acl:` est affecté pour dire `@prefix acl: <https://www.w3.org/ns/auth/acl#> .`

# Fonctionnement

## Conteneurs et ACLs hérités

Le système WAC présume que les documents web sont placés de manière hièrarchique dans des conteneurs ou dossiers. Pour plus de commodité, les utilisateurs n'ont pas a spécifier les permissions sur chaque ressource individuelle -- Ils peuvent simplement fixer des permissions sur un conteneur, ajouter un prédicat `acl:defaultForNew`, et toutes les ressources dans ce conteneur hériteront de ces permissions. 
En d'autres termes, si une Permission contient `acl:defaultForNew`, elle sera appliquée *par défaut* à toutes les ressources dans ce conteneur. 

Vous pouvez écraser les permissions par défaut de n'importe quelle ressource en créant un ACL individuel juste pour la ressource. 

Un exemple d'ACL pour un conteneur : 

```turtle
# Contents of https://alice.databox.me/docs/.acl
@prefix acl: <http://www.w3.org/ns/auth/acl#>.

<#authorization1>
    a acl:Authorization;

    # These statements specify access rules for the /docs/ container itself:
    acl:agent <https://alice.databox.me/profile/card#me>;
    acl:accessTo <https://alice.databox.me/docs/>;
    acl:mode
        acl:Read, acl:Write, acl:Control;

    # defaultForNew says: this authorization (the statements above) will also
    #   be inherited by any resource within that container that doesn't have its
    #   own ACL.
    acl:defaultForNew <https://alice.databox.me/docs/>.
```

**Note :** Le prédicat `acl:defaultForNew` sera bientôt renommé par `acl:default`, à la fois dans la spec et dans l'implémentation des serveurs. La sémantique, comme décrite ici, ne changera pas. 

## ACL de ressource individuelle

Pour un contrôle plus poussé, les utilisateurs peuvent assigner un ensemble de permissions pour chaque ressource individuelle (qui écrase les permissions de son conteneur parent)

#### Exemple d'ACL : 

```turtle
# Contents of https://alice.databox.me/docs/file1.acl
@prefix acl: <http://www.w3.org/ns/auth/acl#>.

<#authorization1>
    a acl:Authorization;
    acl:agent <https://alice.databox.me/profile/card#me>;  # Alice's WebID
    acl:accessTo <https://alice.databox.me/docs/file1>;
    acl:mode
        acl:Read, acl:Write, acl:Control.
```

## Découverte de la location de l'ACL

Prenons une URL pour une ressource individuelle ou contener, un client peut découvrir la location de son ACL correspondant en éxécutant une requête `HEAD` (ou `GET`) et en parsant le contenu du `rel="acl"` du header Link.

Exemple de requête pour découvrir la location d'un ACL pour un document web situé a `http://example.org/docs/file1` : 

```httpspec
HEAD /docs/file1 HTTP/1.1
Host: example.org

HTTP/1.1 200 OK
Link: <file1.acl>; rel="acl"
```

La requête pour découvrir la location d'un conteneur est à peu près similaire : 

```httpspec
HEAD /docs/ HTTP/1.1
Host: example.org

HTTP/1.1 200 OK
Link: <.acl>; rel="acl"
```

Notez que la relation `acl` link utilise des chemins URLs relatifs (Le chemin URL absolu de la ressource ACL au dans l'exemple ci-dessus serait `/docs/.acl`)

**Note dans la [spec](https://github.com/solid/web-access-control-spec#acl-resource-location-discovery)** : 
Les clients NE DOIVENT PAS déterminer la location d'une ressource ACL par dérivation d'une URL d'un document web. Par exemple, prenons un document ayant pour URL `/docs/file1`, les clients ne doivent pas présumer que la ressource ACL est située à `/docs/file1.acl`, simplement en utilisant `.acl` comme suffixe.
La convention de nommage choisie pour les ressource ACL peut être différente pour chaque implémentation individuelle (ou encore chaque serveur). Par exemple, si un serveur trouve la ressource ACL en ajoutant le suffixe `.acl`, un autre serveur pourrait placer les ressources ACL dans un sous-conteneur (dans l'exemple ci-dessus, située a `/docs/.acl/file1.acl`)

## Accès public (Tout les [agents](#description-des-agents))

Pour spécifier un accès public a une ressource, donc donner l'accès a *tout le monde* (par exemple votre profil WebID est en lecture publique), vous pouvez utiliser `acl:agentClass foaf:Agent` pour dire que vous donnez l'accès de la classe à *tout* les agents (le public) 
Par exemple : 

```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#>.
@prefix foaf: <http://xmlns.com/foaf/0.1/> .

<#authorization2>
    a acl:Authorization;
    acl:agentClass foaf:Agent;  # everyone
    acl:mode acl:Read;  # has Read-only access
    acl:accessTo <https://alice.databox.me/profile/card>. # to the public profile
```

## Références aux ressources

Le prédicat `acl:accessTo` spécifie à quelles ressources vous donnez l'accès, utilisant leur URL comme sujets. 

Une ressource ACL étant son propre document web, qu'est ce qui controle *qui* a accès à ce dernier ? Théoriquement, un ACL *pourait* avoir son propre ACL (Donc pour `file1.acl` qui controle l'accès au `file1`, `file1.acl.acl` controlerait potentiellement `file1.acl`), on constate rapidement que cette récursivité doit avoir une fin quelque part. 

C'est là que `acl:Control` des [modes d'accès](#modes-daccès) entre en jeu (voir ci-dessous), il spécifie qui peut modifier (ou même lire) la ressource ACL.

## Modes d'Accès

Le prédicat `acl:mode` dénote une classe d'opérations possibles pour un agent sur une ressource. 

**`acl:Read`**

Donne l'accès en lecture. Dans le node-solid-server, ceci inclut les méthodes HTTP `GET` et `HEAD`. Théoriquement ceci pourrait inclure des systèmes de recherche comme du SELECT en SPARQL. 

**`acl:Write`**

Donne l'accès en écriture sur une ressource, typiquement de la modification de ressource. Dans le node-solid-server, ceci inclut les méthodes HTTP `PUT`, `POST`, `DELETE` et `PATCH`. Ceci inclut donc la possibilité de faire des requêtes SPARQL effectuant des modifications sur les ressource.

**`acl:Append`**

Donne un accès plus limité en écriture sur une ressource -- Ajout seulement. Ceci inclut généralement la méthode HTTP `POST`, et la portion `INSERT` des SPARQL HTTP `PATCH`. Un exemple simple d'utilisation du mode Append, serait une boite mail d'utilisateur -- D'autres agents peuvent écrire (en ajout) des notifications sur la boite mail, mais ne peuvent pas altérer ou lire les autres. 

**`acl:Control`**

C'est un type d'accès spécial qui donne à un agent la capacité de *voir et modifier l'ACL d'une ressource*. Notez que ça n'implique pas que l'agent à un accès `acl:Read` ou `acl:Write` à la ressource cible, mais juste a son document ACL correspondant. Par exemple, un propriétaire d'une ressource pourrait désactiver son accès en écriture pour éviter qu'une application écrase ses données, mais pourrait changer ses niveau d'accès a cette ressource à n'importe quel moment (tant qu'il a un contrôle avec `acl:Control`)


## Description des agents

Dans le WAC, on utilise le terme *Agent* pour identifier *qui* a accès à une ressource. En général, c'est pour dire "Quelqu'un ou quelque chose qui peut être référencé avec une WebID", donc des utilisateurs, groupes (sociétés ou associations), et des agents logiciels comme des applications ou services. 

### Agent unique

Une autorisation peut donner l'accès à n'importe quel nombre d'agents uniques utilisant le prédicat `acl:agent`, et utilisant leur URI de WebID comme objects. [L'exemple de document WAC ACL](#exemple-dacl-) décris précédemment donne l'accès à Alice, comme indiqué par son URI de WebID, `https://alice.databox.me/profile/card#me.`

### Groupes d'agents

Pour donner l'accès a un groupe d'agents, vous pouvez utiliser le prédicat `acl:agentGroup`. L'objet d'un `agentGroup` est un lien vers un document de **Group Listing**. Les WebID des membres du groupes sont listés dedans, utilisant le prédicat `vcard:hasMember`. Si une WebID est listée dans ce document, l'accès lui est donnée.

Exemple de ressource ACL, `shared-file1.acl` contenant une permission de groupe. 

```
# Contents of https://alice.databox.me/docs/shared-file1.acl
@prefix acl: <http://www.w3.org/ns/auth/acl#>.

# Individual authorization - Alice has Read/Write/Control access
<#authorization1>
    a acl:Authorization;
    acl:accessTo <https://alice.example.com/docs/shared-file1>;
    acl:mode acl:Read, acl:Write, acl:Control;
    acl:agent <https://alice.example.com/profile/card#me>.

# Group authorization, giving Read/Write access to two groups, which are
# specified in the 'work-groups' document.
<#authorization2>
    a acl:Authorization;
    acl:accessTo <https://alice.example.com/docs/shared-file1>;
    acl:mode acl:Read, acl:Write;
    acl:agentGroup <https://alice.example.com/work-groups#Accounting>;
    acl:agentGroup <https://alice.example.com/work-groups#Management>.
```

Les Group Lists `work-groups` correspondants : 

```
# Contents of https://alice.example.com/work-groups
@prefix acl: <http://www.w3.org/ns/auth/acl#>.
@prefix vcard: <http://www.w3.org/2006/vcard/ns#> .

<> a acl:GroupListing.

<#Accounting>
a vcard:Group;
  vcard:hasUID <urn:uuid:8831CBAD-1111-2222-8563-F0F4787E5398:ABGroup>;
  dc:created "2013-09-11T07:18:19+0000"^^xsd:dateTime;
  dc:modified "2015-08-08T14:45:15+0000"^^xsd:dateTime;

  # Accounting group members:
  vcard:hasMember <https://bob.example.com/profile/card#me>;
  vcard:hasMember <https://candice.example.com/profile/card#me>.

<#Management>
a vcard:Group;
  vcard:hasUID <urn:uuid:8831CBAD-3333-4444-8563-F0F4787E5398:ABGroup>;

  # Management group members:
  vcard:hasMember <https://deb.example.com/profile/card#me>.
```

**Notes** : 

Je n'ai pas encore testé les groupes de permissions, et il me semble qu'il y a encore quelques bugs pour cette tech. 


## Non supporté par choix

Cette section décrit plusieurs fonctionnalités non implémentées dans la spec par choix. 

#### Propriétaire de la ressource
WAC n'a aucune notion de propriétaire de la ressource, et pas d'accès Propriétaire spécifique. Il est supposé que les Propriétaires sont les agents possèdant les permissions de Read, Write et Control.

#### acl:accessToClass

Le prédicat `acl:accessToClass` n'est pas supporté.

#### Expressions régulières

L'utilisations d'expressions régulières dans des déclarations comme `acl:agentClass` n'est pas supporté. 

