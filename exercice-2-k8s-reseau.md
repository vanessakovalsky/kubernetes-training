# Mise en réseau de notre cluster

Cet exercice a pour objetifs 
* de manipuler les éléments du réseau d'un cluster Kubernetes
* d'exposer un service fonctionnant dans un cluster au reste du monde
* d'explorer les possibilités de la haute disponibilité

## Accéder à un service
* Pour commencer nous allons lancer deux pods :
```
kubectl create deployment multitool --image=praqma/network-multitool
kubectl create deployment nginx --image=nginx:1.7.9
```
### Utilisation de ClusterIP
* Puis nous exposons le pod nginx sur le port 80 avec ClusterIP
```
kubectl expose deployment nginx --port 80 --type ClusterIP
```
* Pour vérifier la liste des services :
```
kubectl get services
```
* Récupérer le nom des pods (pour pouvoir s'y connecter après):
```
kubectl get pods
```
* Cette commande affiche la liste des pods, exemple :
```
NAME                         READY     STATUS    RESTARTS   AGE
multitool-5c8676565d-rc982   1/1       Running   0          3s
```
* Avec le nom du pod nous pouvons lancer un terminal dans le pod concerné
```
kubectl exec -it multitool-5c8676565d-rc982 -c network-multitool -- bash
```
* Depuis le terminal du conteneur du pod (le pod ne fait pour l'instant touné qu'un seul conteneur appelé network-multitool), nous essayons de contacter l'autre pod avec la commande :
```
curl -s 100.70.204.237 | grep h1
```
* Le résultat devrait être :
```
<h1>Welcome to nginx!</h1>
```
* Il est également possible de contacter l'autre pod avec son DNS :
```
curl -s nginx | grep h1
```
* sortir du terminal du conteneur : exit
* -> Notre pod est accessible depuis l'autre pod au sein du même cluster
* Pour obtenir des informations sur un pod comme son adresse ip, la commande describe nous donnes les informations :
```
kubectl describe service nginx
```
* Renvoit :
```
Name:              nginx
Namespace:         default
Labels:            app=nginx
Annotations:       <none>
Selector:          app=nginx
Type:              ClusterIP
IP:                100.70.204.237
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         100.96.1.148:80
Session Affinity:  None
Events:            <none>
```
* Testons depuis l'exterieur :
```
curl -s 100.70.204.237 | grep h1
```
* On obtient alors un accès refusé, ClusterIP permet d'exposer des services entre pods du même cluster, mais ne permet pas d'y accéder depuis l'extérieur

### NodePort

## Haute disponibilité


