Installation Docker
===

### Pour installer le node-solid-server via docker :

1. Télécharger le node-solid-server via github `git clone https://github.com/solid/node-solid-server.git`
2. Le serveur a son propre dockerfile, du coup le build avec `docker build -t node-solid-server .`
3. Le lancer avec `docker run -p 8443:8443 --name solid node-solid-server`

### Pour modifier la configuration : 
* Editez le `config.json-default` ou/et le dockerfile ( notamment la ligne `COPY config.json-default config.json` ) et faire sa config
 
\- OU -
* Suivre la procédure suivante : 
    * Copier la configuration du dossier courant avec `docker cp solid:/usr/src/app/config.json .`
    * Editez le fichier `config.json`
    * Copiez le fichier dans son répertoire d'origine avec `docker cp config.json solid:/usr/src/app/`
    * Redémarrez le serveur avec un `docker restart solid`

#### Notes: 

Voila le dockerfile du node-solid-server : 
```dockerfile=
FROM node:8.11.2-onbuild
EXPOSE 8443
COPY config.json-default config.json
RUN openssl req \
    -new \
    -newkey rsa:4096 \
    -days 365 \
    -nodes \
    -x509 \
    -subj "/C=US/ST=Denial/L=Springfield/O=Dis/CN=www.example.com" \
    -keyout cert.key \
    -out cert.pem
CMD npm run solid start
```

Actuellement, le dockerfile crée un couple cert et key self signed via la commande docker : 
```dockerfile=4
RUN openssl req \
    -new \
    -newkey rsa:4096 \
    -days 365 \
    -nodes \
    -x509 \
    -subj "/C=US/ST=Denial/L=Springfield/O=Dis/CN=www.example.com" \
    -keyout cert.key \
    -out cert.pem
```
utilisés par l'éxectuable solid, et non solid-test, qui unset le flag `NODE_TLS_REJECT_UNAUTHORIZED` et set l'option `rejectUnauthorized`. Cela veut dire que le navigateur bloquera surement les connexion du serveur, si les certificats ne sont pas reconnus comme étant valide. 

Pour résoudre ces problèmes : 

Soit changer la ligne `CMD npm run solid start` par quelque chose comme 
```dockerfile
CMD ./bin/solid-test start
```

Soit mettre les bonnes options pour que les certificats self-signed soient autorisés par le navigateur.
Soit acheter un nom de domaine et le lier a let's encrypt via le certbot