WebID OIDC
===

**Pour contribuer directement à cette documentation cliquez [ici](https://hackmd.io/bMYoCKapToy3IKDDMpXAUA)**

WebID-OIDC est un protocole d'authentification délégué(Authentication delegation protocol), ainsi que des techniques de vérification auth-related. Il est adapté pour des systèmes décentralisés basé sur la technologie WebID, comme [Solid](https://github.com/solid/solid), et comme la plupart des systèmes basés sur LDP. 
Ce protocole est basé sur OAuth2/OpenID Connect décentralisé. 

## Introduction

[Pour en savoir plus sur WebID](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/WebID.md)

Le résultat final de n'importe quel workflow d'authentification basé WebID est une  URI WebID vérifiée (Plus spécifiquement, le bénéficiaire vérifie que l'agent controle l'URI). Par exemple, [WebID-TLS](https://github.com/solid/solid-spec/blob/master/authn-webid-tls.md) dérive le WebID d'un certificat TLS, et vérifie le certificat avec la clé publique dans le profile WebID d'un agent. De manière similaire, la finalité du workflow [OpenID Connect (OIDC)](https://openid.net/specs/openid-connect-core-1_0.html) est un ID token vérifié. Le protocole WebID-OIDC spécifie un mécanisme pour avoir un WebID d'un ID Token OIDC, et gagne les avantages d'à la fois la flexibilité de WebID, et de la sécurité éprouvée d'OpenID Connect.

Regardez aussi : [Les motivations du choix WebID-OIDC](https://github.com/solid/webid-oidc-spec/blob/master/motivation.md)

### Capacités et Avantages

* Authentification cross-domain complêtement décentralisée (n'importe quel peer node peut servir comme identity provider et aussi comme relying party a n'importe quel autre node)
* S'appuie sur des décennies d'expérience de la technologie d'authentification dans le monde réel
* Incorpore des fix aux threat models de: SAML, OpenID et OpenID 2, OAuth et OAuth2. Par exemple : [RFC 6819 - OAuth 2.0 Threat Model and Security Considerations](http://tools.ietf.org/html/rfc6819) -- OpenID Connect a été développé en majeure partie pour résoudre les menaces soulignées ici. 
* Repose sur les épaules de géants (utilise la suite de standards JOSE pour la représentation token, l'enregistrement et l'encryption, comprenant [JWT](https://tools.ietf.org/html/rfc7519), [JWA](https://tools.ietf.org/html/rfc7518), [JWE](https://tools.ietf.org/html/rfc7516) et [JWS](https://tools.ietf.org/html/rfc7515))
* Moyens de déconnexion (et déconnexion unique) (Sign off = déconnexion ?)
* Moyens de [révocation](https://tools.ietf.org/html/rfc7009), blacklist et whitelist d'à la fois les fournisseurs(providers) et applications clients
* Supporte l'authentification de tout les agents et clients : Applications javascript in-browser, server-side web app traditionnelles, applications de bureau et mobiles, et les services IoT
* Compatibilité avec les implémentations de [Web Access Control ACL](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/WacAcl.md#r%C3%A9sum%C3%A9) comme ceux sur les serveurs Solid 
* Met en place l'infrastructure pour ajouter des fonctionnalités a Solid

### Si vous n'êtes pas familier avec OIDC

* Lisez le [résumé de workflow](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/OIDC.md#résumé-de-workflow) plus bas 
* Réferez vous au [Glossaire D'authentification Décentralisée](https://github.com/solid/webid-oidc-spec#decentralized-authentication-glossary) (en)
* Liez [l'article explicatif sur OpenID Connect](https://connect2id.com/learn/openid-connect). Être familier avec les concepts basiques OIDC va être très utile pour comprendre cette spec

## Différences avec OpenID Connect classique

WebID-OIDC fait quelques changement au protocole de base d'OpenID Connect (qui lui même améliore et se construit sur OAuth 2): 
* Débat et formalise l'étape de [Sélection des Fournisseurs](https://github.com/solid/webid-oidc-spec/blob/master/example-workflow.md#21-provider-selection) (provider selection)
* Ajoute la [procédure de dérivation](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/OIDC.md#dériver-un-webid-dun-id-token) de WebID d'un ID Token, vu que les protocoles basés sur WebID utilisent le WebID comme identificateur unique et global (au lieu d'une combinaison de `iss`uer et `sub`ject claims)
* Ajoute une étape additionnelle: [Confirmation de fournisseur WebID](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/OIDC.md#confirmation-du-fournisseur-de-webid) (WebID Provider Confirmation). Après que le WebID a été extraite, le bénéficiaire de l'ID Token doit confirmer que le fournisseur (provider) a bien été autorisé par le détenteur du profile WebID.
* Spécifie le processus de "[Authorized OIDC Issuer Discover](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/OIDC.md#découverte-de-lissuer-oidc-autorisé)" (Utilisé dans le cadre de la confirmation du fournisseur, et durant les étapes de sélection du fournisseur)

Il faut aussi noter : bien que les cas d'utilisation traditionnels d'OpenID Connect sont concernés par récuperer des infos utilisateurs de [UserInfo endpoints](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo), la plupart des systèmes basés sur WebID remplacent le mécanisme UserInfo par le contenu du document de [Profil WebID](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/WebID.md#documents-de-profil-webid)

## Résumé de Workflow

Le workflow d'enregistrement utilisé par le protocole WebID-OIDC : 

Par exemple, voici ce qui se passe quand Alice fait une requête sur la ressource `https://bob.com/resource1`

1. [Requête initiale](https://github.com/solid/webid-oidc-spec/blob/master/example-workflow.md#1-initial-request): Alice (non authentifiée) fait une requête a `bob.com`, reçoit une réponse HTTP `401 Unauthorized`, et on lui présente un écran "Enregistrez vous avec" (Sign in With).
2. [Sélection du fournisseur (Provider Selection)](https://github.com/solid/webid-oidc-spec/blob/master/example-workflow.md#21-provider-selection): Elle sélectionne son fourisseur de service (service provider) en cliquant sur un logo, tapant une uri (par exemple `alice.solidtest.space`), ou en entrant son email.
3. [Authentification locale](https://github.com/solid/webid-oidc-spec/blob/master/example-workflow.md#3-local-authentication-to-provider): Alice est redirigée vers la page d'enregistrement de son fournisseur de service (service provider), `https://alice.solidtest.space/signin`, et s'authentifie en utilisant la méthode proposée de son choix (WebID-TLS, OIDC etc)
4. [Consentement de l'utilisateur](https://github.com/solid/webid-oidc-spec/blob/master/example-workflow.md#4-user-consent) : (Optionnel) Elle peut aussi être présentée devant un écran de consentement de l'utilisateur, avec par exemple les lignes "Voulez vous vous enregistrer à `bob.com` ?"
5. [Réponse de l'authentification](https://github.com/solid/webid-oidc-spec/blob/master/example-workflow.md#5-authentication-response): Elle est ensuite redirigée vers `https://bob.com/resource1` (la ressource qu'elle voulait requêter à la base). Le serveur, `bob.com`, reçoit aussi un ID Token signé de `alice.solidtest.space`, attestant qu'elle s'est authentifiée. 
6. [Dériver un WebID](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/OIDC.md#dériver-un-webid-dun-id-token)(Deriving a WebID URI): `bob.com` (le serveur controllant la ressource) valide l'ID Token, et extrait le WebID d'Alice de l'intérieur du Token. Elle est maintenant enregistrée à `bob.com` comme utilisateur `https://alice.solidtest.space/profile/card#me`.
7. [Confirmation du fournisseur de WebID](https://github.com/assemblee-virtuelle/Doc-Solid/blob/master/fr/OIDC.md#confirmation-du-fournisseur-de-webid)(WebID Provider Confirmation): `bob.com` comfirme que `solidtest.space` est bien le fournisseur OIDC autorisé d'Alice (en comparant avec l'URI du fournisseur dans le `iss` claim du WebID d'Alice)

Il y a beaucoup de processus se passant en interne, effectués par `bob.com` et `alice.solidtest.space`, les deux serveurs concernés dans cet échange. Ils établissent une relation de confiance entre eux (via la [Découverte](https://github.com/solid/webid-oidc-spec/blob/master/example-workflow.md#22-provider-discovery) et l'[Enregistrement Dynamique](https://github.com/solid/webid-oidc-spec/blob/master/example-workflow.md#23-dynamic-client-registration-first-time-only)), ils vérifient leurs signatures respectives avec leur clés publiques, et vérifient l'application client d'Alice (si elle en utilise une). Heureusement, toute cette complexité est cachée de l'utilisateur (et la plupart est aussi cachée au développeur d'applications)

## Dériver un WebID d'un ID Token

Un WebID-OIDC conforme a une Relying Party essaye les méthodes suivantes dans le but d'obtenir un WebID (URI de WebID) d'un ID Token : 

**Méthode 1 - Requête personnalisée de WebID**

En premier lieu vérifier le contenu de l'ID Token pour la déclaration(claim trad needed) `webid`. Cette déclaration est ajoutée a la liste de déclaration de [OpenID Connect ID Token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) par cette doc WebID-OIDC (This claim is added to the set of OpenID Connect ID Token claims by this WebID-OIDC spec TRAD NEEDED)
(Notez que le la liste de déclarations d'ID Token est extensible, intentionnellement, comme expliqué dans la spec core d'OIDC). Si la déclaration(claim) `webid` est présente dans l'ID Token, sa valeur devrait être utilisée comme URI de WebID par le Relying Party, au lieu de la déclaration `sub` traditionnelle.

Cette méthode est utile quand les fournisseur d'identités fournissant les token customisent le contenu de leurs ID Token (et peuvent ajouter la déclaration `webid`);

**Méthode 2 - Valeur de la déclaration `sub` contenant un uri HTTP(s) valide**

Si la déclaration `webid` n'est pas présente dans l'ID Token, le Relying Party devrait vérifier si la déclaration `sub` contient comme valeur un URI HTTP(s). Si la valeur de la déclaration `sub`ject est une URI valide, elle devrait être utilisée comme URI de WebID par le Relying Party.

Cette méthode est utile quand l'identity provider donnant les token ne peut ajouter de déclarations mais peut fixer les valeur de leur propres déclarations `sub`ject (that is, they're not automatically generated as UUIDs, etc).

**Méthode 3 - Requête UserInfo + déclaration`website`

Si un URI de WebID n'est pas trouvé à la fois dans les déclarations `webid` ou `sub`, le Relying Party devrait procéder à une requête [UserInfo](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo) par OpenID Connect, avec l'Access Token approprié qu'il reçoit avec le ID Token. Cette méthode est fournie dans le cas ou les utilisateurs n'ont pas de contrôle sur le contenu des ID Token envoyés par leur fournisseurs(Providers) et ne peuvent donc pas utiliser les méthodes citées précédemment. Cela peut être le cas par exemple si un utilisateur veut s'enregistrer a un Relying Party de WebID-OIDC en utilisant un fournisseur mainstream comme Google. Une fois que la réponse UserInfo est reçue par le Relying Party, la déclaration `website` standard devrait être utilisée par le RP(relying party) comme URI de WebID.

## Confirmation du fournisseur de WebID

#### Le problème

La spec OIDC utilise l'ID Token comme id utilisateur unique la déclaration `sub`ject. Seulement, cela requiert que l'id est unique *pour un fournisseur donné*. En prenant en compte que le WebID est *globallement* unique comme identification d'utilisateur, le protocole WebID-OIDC doit faire une étape en plus et *confirmer* que la personne possèdant le WebID a autorisé un fournisseur donné(a given provider) à utiliser ce WebID. Sinon la situation suivante peut se produire : 

1. Alice s'enregistre à `bob.com` avec son identity provider de son choix, `alice.com`. L'ID Token pour `alice.com` déclare que le WebID d'Alice est `https://alice.com/#i`, jusque là tout va bien.
2. Un hacker s'enregistre aussi à `bob.com`, en utilisant comme identity provideur `evilbox.com`. Et parce qu'il se trouve qu'il controle ce serveur, il peut mettre ce qu'il veut dans la déclaration `webid` de l'ID Token sortant de ce serveur. Du coup ils peuvent *aussi* dire que leur `webid` est `https://alice.com/#i`

#### La solution

Quand présenté avec les identifiants(credentials) WebID-OIDC sous la forme de bearer tokens, le serveur de ressource DOIT confirmer que l'identity provider (la valeur de la déclaration `iss`uer) est autorisée par détenteur du WebID en faisant les choses suivantes : 

1. (Le plus courant) Si le serveur fournissant l'ID Token est la même entitée que celui qui accueille le profil WebID, alors il est considéré comme le fournisseur OIDC autorisé pour cette WebID (court circuite le processus de Provider Confirmation), et aucune étape supplémentaire n'est nécéssaire. Spécifiquement, l'un des suivants doit être vrai:
        - [L'origine](https://developer.mozilla.org/en-US/docs/Web/API/URL/origin) de l'URI de WebID est la même que l'origine de l'URI de la déclaration `iss`uer. (Par exemple, `iss:'https://example.com'` et l'URI de WebID est `https://example.com/profile#me`).
        - L'URI de WebID est un *sous domaine* de l'issuer du ID Token. Par exemple, `iss:'https://example.com'` et l'URI de WebID est `https://alice.example.com/profile#me`. Si aucun des deux n'est le cas (et que le WebID est hébergée dans un domaine de sécurité différent que son provider OIDC), des étapes supplémentaires sont nécéssaires poru la confirmation.
2. Déterminer le provider d'URI OIDC autorisé pour cette WebID, en faisant une étape de [Découverte de l'issuer OIDC autorisé](#).
3. Si l'URI du Provider n'est pas découvrable, soit par le header ou le body du [profile WebID](https://github.com/solid/solid-spec#webid-profile-documents), le serveur de ressource DOIT rejeter les crédentials ou les tentatives d'authentification. 
4. Si l'URI du Provider est découverte, elle DOIT être égale à l'URI de l'issueur dans l'ID Token (la déclaration `iss`uer) et rejeter les credentials dans le cas contraire. 

## Découverte de l'Issuer OIDC autorisé

Durant les étapes de sélection du Provider ou de la [confirmation du Provider](#confirmation-du-fournisseur-du-webid), il est nécéssaire de découvrir, pour un WebID donné, l'URI du provider OIDC autorisé pour ce WebID.

**Découverte de l'issuer par le header `Link`**

Pour découvrir l'issuer OIDC autorisé pour un WebID par les header Link rel : 
1. Faire une requête HTTP OPTION à l'URI de WebID
2. Parser le header `Link:` et vérifier la valeur de la relation link`http://openid.net/specs/connect/1.0/issuer`. Si elle est présente, utiliser la valeur de cette relation link comme le provider d'URI autorisé. Par exemple `Link:<https://provider.example.com>; rel="http://openid.net/specs/connect/1.0/issuer"` signifie que `https://provider.example.com` est le provider OIDC autorisé pour cette URI.
3. Si le header Link n'est pas présent (ou ne contient pas la relation Link), il faut aller à l'étape suivante, soit la découverte de l'issuer par le contenu du profile WebID.

**Découverte de l'issuer par le profile WebID**

Pour découvrir l'issuer OIDC autorisé pour une WebID par le contenu du profile WebID (ceci requiert du parsing Turtle/RDF):

1. Déréferencer l'URI de WebID (faire une requête GET HTTP) et chercher le contenu du profile WebID (un fichier turtle ou json-ld ou autre format RDF).
2. Parsez le RDF, et chercher pour l'objet de l'instruction contenant le prédicat `<http://www.w3.org/ns/solid/terms#oidcIssuer>`.

Par exemple, si Alice (avec le WebID `https://alice.example.com/profile#me`) veut spécifier comme provider OIDC autorisé `https://provider.com` pour ce profil, elle devrait ajouter les triplets suivant a son profil : 

```turtle
@prefix solid: <http://www.w3.org/ns/solid/terms#>.

# ...

<#me> solid:oidcIssuer <https://provider.com> .
```
