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
kubectl set image deployment nginx nginx=nginx:1.24.0
kubectl annotate deployment nginx kubernetes.io/change-cause="version change from latest to 1.24.0 - training purpose" --overwrite=true
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
 1.25.1, 1.24.3, 1.26.0
* Vérifier ce qu'il se passe avec :
```shell
$ kubectl get pods
```

### Annuler une mise à jour 
* Faire une mise à jour avec une image qui n'existe pas :
```shell
kubectl set image deployment nginx nginx=nginx:100.200.300
kubectl annotate deployment nginx kubernetes.io/change-cause="version change to broken version" --overwrite=true
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
