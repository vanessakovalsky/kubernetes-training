# Deploiement de pods, mises à l'échelle et vérification

Cet exercice a pour objectfs :

* De vous permettre de mettre à l'échelle vos pod dans votre cluster
* De faire des mises à jours de déploiements

## De vous permettre de mettre à l'échelle vos pod dans votre cluster
Nous allons travailler sur la scalabilité de notre pod, afin de permettre d'augmenter le nombre de replicas disponibles.$

* Commencer par augmenter le nombre de replicas disponible à 4 pour notre déploiement nginx :
```shell
kubectl scale deployment nginx --replicas=4
deployment.extensions/nginx scaled
```
* Vérifier le résultat sur les déploiements et les pods :

```shell
kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
multitool   1/1     1            1           24m
nginx       4/4     4            4           34m
```

```shell
kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          24m
nginx-569477d6d8-4msf8       1/1       Running   0          20m
nginx-569477d6d8-bv77k       1/1       Running   0          34s
nginx-569477d6d8-s6lsn       1/1       Running   0          34s
nginx-569477d6d8-v8srx       1/1       Running   0          35s
```

**NB:** Le déploiement nginx dit Ready=4/4, Up-to-date=4, Available=4. Les pods montre la même chose, il y a maintenant 4 pods nginx qui fonctionnenet. L'un d'eux est le premier a avoir été lancer, et les autres ont été créé quelques secondes auparavant par notre commande précédente.

* Il est également possible de réduire l'échelle, par exemple à 2 :

```shell
$ kubectl scale deployment nginx --replicas=2
deployment.extensions/nginx scaled
```
* On vérifie le résultat :
```shell
kubectl get pods
NAME                         READY     STATUS        RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running       0          25m
nginx-569477d6d8-4msf8       1/1       Running       0          21m
nginx-569477d6d8-bv77k       0/1       Terminating   0          1m
nginx-569477d6d8-s6lsn       0/1       Terminating   0          1m
nginx-569477d6d8-v8srx       1/1       Running       0          2m
```
* Les pods non utilisé sont tués immédiatement :

```shell
kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          26m
nginx-569477d6d8-4msf8       1/1       Running   0          22m
nginx-569477d6d8-v8srx       1/1       Running   0          2m
```

## Faire des mises à jours de déploiements
* Vériier que le service de load balancing tourne toujours :
```shell
kubectl get service
```
* Remettre 4 replicas que nous allons utiliser pour le déploiement
```shell
kubectl scale deployment nginx --replicas=4
```

### Mise à jour du déploiement

* Mettre à jour l'image utilisé pour le déploiement 

```shell 
kubectl set image deployment nginx nginx=nginx:1.9.1 --record
```
* Vérifer le statut du rollout
```shell
kubectl rollout status deployment nginx
```
* Voir l'historique des mises à jour faite par rollout :

```shell
kubectl rollout history deployment nginx
```
* Essayer de répeter la mise à jour de l'image avec la commande set image. Quelques suggestions de versions d'images utilisables :
 1.12.2, 1.13.12, 1.14.1, 1.15.2.
* Vérifier ce qu'il se passe avec :
```shell
$ kubectl get pods
```

### Annuler une mise à jour 
* Faire une mise à jour avec une image qui n'existe pas :
```shell
kubectl set image deployment nginx nginx=nginx:100.200.300 --record
```
* Cela entraine le non fonctionnement de certaint pods
* On peut vérifier ce qu'il s'est passé avec :
```shell
kubectl rollout history deployment nginx
```
* Pour annuler (défaire) le déploiement et restaurer la version précédente :
```shell
kubectl rollout undo deployment nginx
```
* Vérifier les pods en cours d'execution :
```shell
$ kubectl get pods
```

