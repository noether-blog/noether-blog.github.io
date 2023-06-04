---
author: froggy
title: Mais au fait un orchestrateur c'est quoi ?
date: 2023-06-04 02:00:00 +0800
categories: [Blogging, Ops, Tooling]
tags: [tooling, sre, ops]
toc: false
comments: false
mermaid: true
---

Kubernetes, Nomad, Openshift, ils se font appeler sous pleins de noms mais restent 
un mystère pour pas mal de monde. 

L'objectif avec cet article est de démystifier tout cela et comprendre ce qu'est un orchestrateur,
à quoi ça sert, comment ça fonctionne et à quelles problématiques on cherche à répondre avec.

> Pré-requis: Connaitre le vocabulaire docker, conteneur / image

## Tout commence dans un entrepôt 

Commençons simplement, imaginons un entrepôt dans lequel on dépose des colis sur des étagères.

- Chaque colis possède un poids et un volume;
- Chaque étagère se voit un poids et un volume maximal qu'elle peut accueillir;
- Les colis n'ont rien d'unique et sont associés à une référence produit en fonction de leur contenu, 
deux colis peuvent donc avoir une référence produit commune.

Le but du jeu est de répartir vos colis sur l'ensemble des étagères de votre entrepôt 
en respectant les contraintes suivantes:

- La somme des poids des colis sur une même étagère ne doit pas dépasser le poids maximal supporté par l'étagère;
- La somme des volumes des colis sur une même étagère ne doit pas dépasser le volume maximal supporté par l'étagère;
- Tous les colis d'une même référence produit ne doivent pas être sur la même étagère.

On pourrait également rajouter des contraintes un peu personnalisées, 
par exemple une étagère pourrait ne prendre que les colis bleus.

Une fois vos colis bien placés, les employés de l'entrepôt doivent pouvoir les retirer pour charger un camion par exemple,
pour une référence produit donnée vous devez donc être capable de donner l'emsemble des étagères sur lesquelles 
les colis correspondant ont été placés dans l'entrepôt afin qu'un employé puisse en choisir un à retirer.

Vous devrez également assurer une disponibilité permanente, une référence produit est 
associée à un certains nombre de colis à positionner dans l'entrepôt, le nombre de colis 
disponible dans l'entrepôt doit toujours rester le même, si un colis est retiré d'une étagère 
vous devrez alors réaprovisionner votre entrepôt en quantité nécessaire pour avoir le bon nombre de colis.

## Ok t'es gentil, mais les orchestrateurs dans tout ça ?!

Et bien ce que vous avez fait avec l'entrepôt c'est exactement ce qu'un orchestrateur 
comme Kubernetes va faire avec votre parc applicatif.

Dans votre cluster (*entrepôt*):

- Notre application (*référence produit*) requiert des resources cpu et ram pour fonctionner (*poids et volumes*);
- Chaque nodes/server (*étagère*) de votre cluster, possède une certaine quantité de cpu et de ram disponible;
- Pour chaque application on va déployer un conteneur (*colis*) sur plusieurs nodes en suivant les mêmes contraintes de l'entrepôt;
- Quand je cherche à appeler notre application il faut savoir où rediriger ma requête pour qu'elle soit traitée par le conteneur correspondant;
- Le nombre de conteneur déployé pour mon application est défini en amont et doit toujours être le même.

Par confort j'ai fait un raccourci en disant "Une application = Un conteneur", 
mais une application peut très bien correspondre à plusieurs conteneurs travaillant de concert.

## Exemple avec Kubernetes

### La théorie

Pour travailler avec kubernetes il faut désormais se 
plonger un peu plus dans son fonctionnement

Le schéma ci-dessous montre ce qu'il se passe lorque je fais un appel 
à une application hébergée sur kubernetes depuis mon navigateur web

```mermaid
flowchart LR
    F([Navigateur]) --> |Appel web| A
    subgraph Kube Cluster
        A[Ingress] -->|Call service| B[App Service]
        B -->|Load Balance| D[App First Pod]
        B -->|Load Balance| E[App Second Pod]
    end
```

Au niveau du Kube Cluster on retient les 3 termes suivants

- Ingress
- Service
- Pod

Le `pod` pour reprendre l'exemple de l'entrepôt correspond à notre *colis*,
c'est le pod qui contient les conteneurs faisant tourner votre application.
Pour votre application vous avez le choix de déployer autant de pod que vous
souhaitez (on parle de `replicas`)

Le `service` est une entité propre à kube qui va se charger de dispatcher l'ensemble
des appels envoyés à votre application sur tous les pods de votre application 
que vous aurez déployé.

Avant de continuer, il faut préciser que tous vos pods et services sont coupés du monde 
extérieur et ne peuvent pas être appelé directement. 

C'est là que l'`ingress` entre en jeu, kubernetes s'il est configuré pour peut posséder un
pod par défaut appelé `ingress-controller`, ce pod a la particularité d'avoir un pied dans 
kube et en dehors de kube.

L'ingress reçoit donc l'intégralité des appels du monde extérieur, pour chacun de ces appels,
son rôle sera de déterminer à quel service il correspond puis de le rediriger dessus, le service
se chargera alors de rediriger l'appel sur un des pods faisant tourner notre application.

### Demo

Pour notre exemple je vais prendre l'exemple d'une simple api python.

- L'api écoute sur le port 8080 
- Au démarrage de l'api on génère un identifiant d'instance unique propre à notre api
- On expose un endpoint `/api/v1/sum/<num1>/<num2>` qui retourne dans un json la somme de num1 et num2 et l'identifiant d'instance de notre api

Pour `/api/v1/5/7` si l'identifiant d'instance de l'api est `noether-23031882` le résultat sera

```json
{
  "result": "12",
  "instance_id": "noether-23031882"
}
```

Le code de cet api avec son dockerfile est disponible [ici](https://github.com/noether-blog/python-sum-api)

Après avoir cloné le projet, je construis une image docker de mon application python

```bash
docker build -t sum-api:v1.0.0 .
```

Je possède désormais une image de mon application dont je peux lancer un conteneur avec la commande suivante

```bash
docker run --rm -p 8080:8080 sum-api:v1.0.0
```

Dans un terminal à part on lance la commande

```bash
curl http://localhost:8080/api/v1/sum/15/23` 
```

Et l'on observe le résultat suivant (votre `instance_id` doit être différent du miens)

```json
{
  "instance_id": "WChAEYJMLzhOnzQGI1J2d8xQ8",
  "result": "38"
}
```

L'instance id est une chaine de charactère générée aléatoirement par l'application qui sera toujours la même pour cette instance.

Si je lance dans un nouveau terminal un nouveau conteneur de mon api mais cette fois sur le port 4200

```bash
docker run --rm -p 4200:8080 sum-api:v1.0.0
```

Que j'appelle la même api sur ce nouveau conteneur

```bash
curl http://localhost:4200/api/v1/sum/15/23
```

Alors j'obtiens un `instance_id` différent

```json
{
  "instance_id": "XlC7sbAJI4FnS4NLi4C1KuP4P",
  "result": "38"
}
```

On peut couper les conteneurs (`Ctrl + C` dans le terminal associé) si vous avez suivi mes commandes ils seront supprimés automatiquement

Kubernetes étant un très gros logiciel, on ne peut pas directement travailler avec sans mettre son pc à genoux, 
on va donc utiliser une version lite de celui-ci adapté pour desktop, j'ai nommé [k3s](https://k3s.io/), 
rendez vous sur le site et suivez l'installation

## Pourquoi on fait ça ?

Pour une seule et unique raison ! Vos pods sont déployés sur plusieurs machines (appelées noeuds ou nodes)
Cela vous permet donc plusieurs choses

Traiter plusieurs appels à votre api en parallèle, le service agissant en temps que load balancer
va dispatcher le traffic entrant sur l'ensemble de vos pods.

Assurer la résilience de votre application, si un noeuds doit redémarrer pour maintenance, les autres
sont toujours là pour pouvoir traiter les requêtes entrantes.

Déployer une nouvelle version de votre application sans interuption de service. 
Pendant que votre nouvelle version est déployée sur un noeud et non disponible, 
les autres noeuds prennent le relais, puis une fois déployées et démarrées sur un noeud, 
on fait la mise à jour sur les autres noeuds.

## Contraintes de développement

Avoir l'ensemble de votre infrastructure tournant sur un orchestrateur implique donc
de se poser plusieurs questions quant à notre façon de développer et de concevoir notre système.

Certains choix d'architecture peuvent se faire par confort (event driven, micro-service) mais
ne sont en rien une obligation.

Partant du principe que si votre application tourne sur plusieurs noeuds, cela veut dire que sur deux appels consécutifs
le traitement pourra se retrouver sur deux noeuds différents; auquel cas le deuxième traitement n'aura pas de connaissance
directe de ce qu'à fait le premier traitement.

La plus grande contrainte à respecter est donc de penser vos applications en stateless, votre application doit être capable
lorsqu'elle reçoit un appel de déterminer l'ensemble des infos qui ont amené cet appel sans connaitre le passé.

## Conclusion

C'en est fini pour cet article sur les orchestrateurs, j'espère qu'il vous aura permis de démystifier 
le sujet et d'y voir un peu plus clair sur le fonctionnement de ces outils.

L'exemple est centré sur kubernetes car c'est l'un des plus populaires, mais on retrouve le même principe sur les autres.
