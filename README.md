# NightClazz Elastic - Zenika Lille

Durant cette NightClazz, nous allons mettre en place une 
infrastructure de gestion des logs (Apache). Nous allons
essayer de mettre en place le maximum de briques, en fonction
du temps restant. 

Le but final sera d'avoir accès à des graphiques permettant 
de visualiser vos logs, les manipuler, les filtrer, ... et 
de vous alerter dès qu'une anomalie est détectée. 

Si vous êtes en avance, n'hésitez pas à jeter un coup d'oeil aux 
autres produits et autres fonctionnalités de la stack Elastic. 

Prérequis : 
* Java 8

## Lancement de l'application

Pour générer des logs, nous allons utiliser une application que nous 
avons développer pour l'occasion. C'est une application web, permettant
des générer des logs dès qu'une requête est envoyée. Une requête vers une
ressource statique inexistante a été ajoutée pour générer une erreur 404.  

Pour lancer l'application, vous devez exécuter la commande suivante : 

```shell
java -jar application.jar
```

## Récupération des logs

Pour la récupération de chaque ligne de log nous allons utiliser 
**FileBeat** . Cet agent s'installe sur les serveurs sur lesquels
vous désirez récupérer les logs pour les envoyer quelque part. 

Si vous avez des doutes, nous vous invitons à regarder la documentation 
officielle : https://www.elastic.co/guide/en/beats/filebeat/current/index.html

Suivez les étapes ci-dessous afin de mettre en place cet outil. 

* Téléchargez **Filebeat** : https://www.elastic.co/downloads/beats/filebeat
* Dezippez l'archive téléchargée
* A partir du fichier `filebeat.yml`, créez un fichier `nightclazz.yml` avec les caractéristiques suivantes : 
    * Lire le fichier access_log situé dans le répertoire logs généré par l'application
    * Chaque ligne sera de type `access_log` (propriété `document_type`)
    * A chaque ligne lue, ajouter le champs `application` avec la valeur `nightclazz` grâce à la propriété `fields`

Nous allons utiliser pour l'instant la sortie **Console**. 
* Ajoutez le paramètre `pretty` afin de faciliter la visualisation des documents

Nous sommes à présent prêt pour lancer **Filebeat** afin de tester notre configuration. 
* Exécuter la commande suivante : 

```shell
./filebeat -c nightclazz.yml
```

Vous devriez apercevoir dans votre console de nouveaux documents JSON à chaque raffraichissement de l'application. 
Si c'est le cas, nous pouvons passer à l'étape suivante...

## Traitement des logs

Pour traiter les logs lues par **Filebeat**, nous allons mettre en place un **Logstash**. Le but de cette brique
sera de découper la propriété `message` du document retourné par **Filebeat** afin d'extraire les informations intéressantes. 

* Téléchargez **Logstash** : https://www.elastic.co/fr/downloads/logstash
* Dezippez l'archive téléchargée
* Dans le répertoire de **Logstash**, créez un fichier `nightclazz.conf` dans lequel nous allons définir la configuration de l'outil. 

Pour tester la configuration, nous allons faire un pipeline très simple. Pour chaque chaine de caractère insérée (**input**), nous 
retournons le document générée dans la sortie standard (**output**).

* Configurez **Logstash** pour utiliser l'input **stdin** et l'output **stdout**
* Lancez **Logstash** via la commande suivante

```shell
./bin/logstash -f nightclazz.conf
```

Maintenant que Logstash est installé, nous allons pouvois le configurer afin de traiter les documents
lus et envoyés par **filebeat**

* Configurez **filebeat** pour envoyer le document au **logstash** que nous venons d'installer
    * L'index dans lequel seront situés les logs est l'index **logs**
* Configurez **logstash** pour pouvoir récupérer en entrée los logs retournées par **filebeat**
    * Pour cela, vous devez configurer l'input **beats**
* Si nous relançons **filebeat** et **logstash**, vous devez normalement voir dans la console de **logstash**
toutes les logs générées à chaque rafraichissement de l'application.

Maintenant nous allons découper la propriété `message` de chaque document que nous recevons. 

* Ajouter un filtre `grok` permettant de découper le message. pour tester le pattern `grok`, vous pouvez
utiliser le site :  http://grokconstructor.appspot.com/do/match. Vous trouverez la liste des patterns groks 
disponibles sur le github de **logstash** : https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns
* Synchroniser la propriété `@timestamp` avec la date présente dans la log. (utilisation du filtre `date`)
* Supprimer la propriété `message` de chaque document.

Avec cette configuration vous devriez voir dans la console **logstash**, une document **JSON** contenant
les différentes propriétés que vous avez défini dans le filtre **grok**.

## Indexation dans ElasticSearch

Voilà, nous avons nos logs... Il est temps de les envoyer dans ElasticSearch afin de pouvoir faire des 
recherches et de pouvoir visualiser ces données dans Kibana. 

* Téléchargez **ElasticSearch** : https://www.elastic.co/fr/downloads/elasticsearch
* Dezippez l'archive téléchargée
* Lancez votre instance via la commande suivante 

```shell
./bin/elasticsearch
```

* Modifiez le fichier de configuration afin d'envoyer nos logs à notre noeud ElasticSearch
* Via une requête REST, veuillez vous assurer que les documents sont bien reçus par ElasticSearch.

Avant de faire d'autres reqquêtes, nous allons tout d'abord modifier le mapping de notre index, 
afin d'optimiser les résultats envoyés par ElasticSearch. Veuillez respecter les contraintes suivantes : 

* `auth` est une chaîne de caractères non analysée
* `bytes` doit être un entier
* `durationMs` doit être un entier
* `clientip` est une chaîne de caractères non analysée
* `host` est une chaîne de caractères non analysée
* `httpversion` est une chaîne de caractères non analysée
* `ident` est une chaîne de caractères non analysée
* `reponse` est une chaîne de caractères non analysée
* `request` est une chaîne de caractères non analysée

Maintenant que le mapping est correctement configuré, nous pouvons à présent exécuter nos premières requêtes : 

* Combien de document avez-vous dans votre index
* Combien de requêtes avons nous avec un code de retour égal à 404
* Retournez tous les documents contenant la chaîne de caractère `angular`
* Retournez tous les documents dont la propriété `request` contient la chaîne de caractère `angular`. Pourquoi avez-vous ce résultat ? 
* Modifiez le mapping de l'index afin de pouvoir exécutez la requête précedente
* Retournez toutes les requêtes dont la propriété `durationMs` est supérieur à 50
* Récupérez toutes les logs indexées depuis 5 minute
* Pour chaque valeur de la propriété `request`, calculez le temps de moyen nécessaire pour envoyer la réponse. 
* Retournez des statistiques sur le temps de réponses de chaque requête (proriété `durationMs`)
* Créez un template afin de configurer pour vos futurs indexes **logstash-*** le mapping de vos documents

A votre tour à présent d'imaginez d'autres requêtes...

## Visualisation dans Kibana

* Téléchargez **Kibana** : https://www.elastic.co/fr/downloads/kibana
* Dezippez l'archive téléchargée
* Lancez votre instance via la commande suivante 

```shell
./bin/kibana
```

Vous dezvez à présent avoir Kibana accessible à l'URL `localhost:4601`

Dans Kibana, nous allons bien evidemment manipuler les logs indexées précédemment. Pour cela, veuillez
configurer Kibana afin de pouvoir accéder aux logs de l'index **logstash-***

* Découvrer l'onglet **Discover** 
    * Affichez que les requêtes avec le code de retour 404
    * Affichez que les requêtes contenant la chaine de caractère `angular`
    * Dans le tableau, affichez la colonne `durationMs`, `request`, `response` et `bytes`
    * Sauvegardez cette nouvelle recherche.

Vous pouvez à présent créer vos propres visualisations, la seule limite reste votre imagination. Voici des exemples de widget que vous vous 
pouvez créer dans Kibana :

* une visualisation de type métrique avec le nombre de logs indexées
* un histogramme représentant le nombre de logs en fonction du temps
* un camembert représentant la répartition des codes de retour (200, 404, ...)
* un graphe de type `line chart` représentant le temps moyen pour l'ensemble des requêtes

* Veuillez à présent agréger les différentes visualisations et recherches dans un dashboard, et découvrez les intéractions disponibles. 


## Envoyez moi des alertes svp !!!

Pour terminer cette NightClazz, nous allons mettre en place un système d'alerting. Pour cela, nous allons 
utiliser le module d'Alerting proposé par Elastic, et également le service **ifttt**  et son module **Maker**. 

Le module **Maker** permet d'exécuter une action compatible avec **ifttt** (mail, tweet, ...) lorsque il
recoit une requête que nous avons configuré. 

- Connectez vous à **ifttt**
- Créez une nouvelle Applet : 
    - écoutant l'événement `elastic-alert` via le module **Maker**
    - exécutant l'action que vous désirez (envoie d'un tweet, envoie d'un email, ...)
- Testez cette nouvelle Applet pour vérifier son bon fonctionnement grâce à la page de test disponible : https://internal-api.ifttt.com/maker

- Arrêtez **ElasticSearch** et **Kibana**
- Installez le pack **X-PACK** à votre noeud **ElasticSearch** et à **Kibana**

```
elastic/bin/elastic-plugin install x-pack
kibana/bin/kibana-plugin install x-pack
```

- Redémarrez **ElasticSearch** et **Kibana** (une connextion sera nécessaire. Login: elasticsearc, password: changeme)

- Créez une alerte permettant d'envoyer une requêtes vers **Maker** dès que le nombre de réponse 404 est supérieur à 5 dans
les dernières 5mn.
