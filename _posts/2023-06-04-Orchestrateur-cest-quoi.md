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

#### Présentation de notre application

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

#### Mise en place de notre cluster kubernetes

Kubernetes étant un très gros logiciel, on ne peut pas directement travailler avec sans mettre ses ressources pc à mal, 
on va donc utiliser une version lite de celui-ci adapté pour desktop, j'ai nommé [k3s](https://k3s.io/), 
rendez vous sur le site et suivez la procédure d'installation.

Vous devez désormais avoir un service `k3s` qui tourne sur votre machine, vous devriez pouvoir le consulter via la commande suivante

```bash
sudo systemctl status k3s.service
```

Nous allons devoir modifier un petit peu le service k3s pour travailler avec afin qu'il soit capable de retrouver
notre image docker de notre api python. Pour cela ouvrez le fichier suivant en tant qu'administrateur `/etc/systemd/system/k3s.service`

À la dernière ligne, rajoutez `--docker` comme suit, vous indiquerez donc à k3s d'utiliser votre docker comme moteur de conteneur

```txt
ExecStart=/usr/local/bin/k3s \
    server \
    --docker
```

Pour redémarrer le service lancer les commandes suivantes

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s.service
```

> Les commandes qui suivent sont à lancer avec `sudo`

Une fois l'installation terminée on va créer un namespace `noether` dans lequer travailler

```bash
kubectl create namespace noether
```

> Un namespace est un cloisement interne à kube dans lequel on choisi de ranger nos applications, deux applications dans deux namespaces différents
seront alors complètement isolée l'une de l'autre.

#### Déploiement de notre application - Pod et Deployment

Petit rappel de vocabulaire dans kubernetes notre application tourne dans un `pod` pour créer un pod kubernetes
possède un objet nommé `deployment` qui est un descriptif de comment créer un pod que kubernete utilisera, 
si nous souhaitons faire tourner notre application il faut donc créer un déploiement pour notre image docker.

La commande suivante vous permettra de voir à quoi ressemble un `deployment`, le paramètre `--dry-run=client` indique à kube de ne rien opérer

```bash
kubectl create deployment py-sum-api --image=sum-api:v1.0.0 --dry-run=client -o yaml -n noether
```

Vous pouvez sauvegarder ce yaml en rajoutant `> manifest-deployment.yaml` à la suite de votre commande

```bash
kubectl create deployment py-sum-api --image=sum-api:v1.0.0 --dry-run=client -o yaml -n noether > manifest-deployment.yaml
```

Le manifest est disponible [ici](https://github.com/noether-blog/python-sum-api/blob/main/k3s/manifest-deployment.yaml) 

Pour déployer notre application on a désormais deux commandes possibles

```bash
kubectl create deployment py-sum-api --image=sum-api:v1.0.0 -o yaml -n noether
kubectl apply -f manifest-deployment.yaml
```

Une fois une de ces deux commandes lancées vous pouvez consulter votre déploiement et le pod qui a été créé à l'aide des deux commandes suivantes

```bash
kubectl get deployment -n noether # Va nous afficher la liste des déploiements
kubectl get pods -n noether # Va nous afficher la liste des pods
```

> Si vous n'êtes pas à l'aise avec la ligne de commande je vous invite à regarder le logiciel [lens](https://k8slens.dev/)

Ici notre déploiement n'a créé qu'un seul pod. Pour notre démo on aimerait en avoir deux

Dans `manifest-deployment.yaml` nous allons nous intéresser au paramètre `spec.replicas`, ce paramètre va indiquer à kube combien de pods il doit créer

```yaml
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
```

On édite le manifest pour passer le nombre de replicas à deux et on apply de nouveau le manifest

```bash
kubectl apply -f manifest-deployment.yaml -n noether
```

> On aurait également pu utiliser la commande `kubectl edit deployment py-sum-api` pour faire l'édition en live

Si vous relancez la commande `kubectl get pods -n noether` vous aurez alors un résultat similaire à celui-ci

```txt
NAME                          READY   STATUS    RESTARTS   AGE
py-sum-api-5779b44f4b-65bxr   1/1     Running   0          87s
py-sum-api-5779b44f4b-2bmm6   1/1     Running   0          45s
```

Nous aimerions désormais pouvoir accéder à ces pods via notre navigateur.

#### Déploiement de notre application - Les Services

Dans Kubernetes les pods sont complètements isolés du monde extérieur, et n'ont pas de possibilité 
d'être appelé directement, d'autant plus que si c'était le cas, comment savoir lequel vous allez appelé ? 
C'est là que les services entre en jeu.

Le `service` est une entité kube qui (dixit la doc) définit un ensemble logique de pod ainsi que des règles pour y accéder.

Le `service` se place donc devant l'ensemble de vos pods et se charge de dispatcher le traffic dessus selon des règles établies.

Notre service comme un deployment peut être représenter par un manifest yaml, 
on peut le consulter avec la commande suivante, comme précédemment le `--dry-run=client` ne vas pas exécuter la commande

```bash
kubectl expose deployment/py-sum-api --type="NodePort" --port 8080 --dry-run=client -o yaml -n noether
```

On peut donc sauvegarder ce manifest en rajoutant `> manifest-service.yaml` à la suite de la commande

```bash
kubectl expose deployment/py-sum-api --type="NodePort" --port 8080 --dry-run=client -o yaml -n noether > manifest-service.yaml
```

> Nous reviendrons sur le NodePort après.

Vous pouvez donc désormais créer votre service avec la commande suivante

```bash
kubectl apply -f manifest-service.yaml -n noether
```

Vous pouvez désormais lister vos services avec la commande `kubectl get services -n noether` et vous aurez un résultat similaire à celui ci-dessous

```txt
NAME         TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
py-sum-api   NodePort   XX.YY.ZZ.ABC   <none>        8080:30784/TCP   3m2s
```

Si vous prenez la `Cluster-ip` de votre service et vous rendez dans votre navigateur sur l'url `http://xx.yy.zz.abc:8080/health`

Vous obtiendrez le json suivant, correspondant au [endpoint](https://github.com/noether-blog/python-sum-api/blob/main/app.py#L14) `/health` de notre api python

```json
{
  "status": "ok"
}
```

Appelons désormais l'url suivante plusieurs fois `http://xx.yy.zz.abc:8080/api/v1/sum/15/7`

De mon côté j'observe les deux résultats suivants

```json
{
  "instance_id": "B6mG03Ptg9nbWylnf72d9uIMs",
  "result": "22"
}
```
```json
{
  "instance_id": "AcDk7Y8M3VA12keqaKWf90taz",
  "result": "22"
}
```

On constate que l'`instance_id` est différent c'est parque le service a dispatché l'appel http sur les deux pods.

Je vous ai parlé du `NodePort` un service de type NodePort va exposer le port 8080 du noeuds sur lequel tournent les pods directement au monde extérieur.

Ce n'est pas spécialement une bonne pratique que d'exposer tous les nodes de son cluster à l'extérieur, 
on préféra utiliser des services de type `ClusterIP` rendant les pods accessibles uniquement depuis le cluster

Pour exposer à l'extérieur vous aurez donc un seul service de type `NodePort` qui recevra tout le traffic en entrée pour le transmettre à un pod qui sera alors
chargé de dispatcher le traffic à tous les autres services du cluster. On parle alors de ce pod comme étant un `ingress`.

## Pourquoi fait on ça ?

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


## Source pour cet article

[la documentation kubernetes](https://kubernetes.io/docs/home/)
