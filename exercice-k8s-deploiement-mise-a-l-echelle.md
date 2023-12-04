# Deploiement de pods et mises à l'échelle 

Cet exercice a pour objectfs :

* De déployer un pod de manière replica
* De vous permettre de mettre à l'échelle vos pod dans votre cluster

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


