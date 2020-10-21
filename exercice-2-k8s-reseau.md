# Mise en réseau de notre cluster

Cet exercice a pour objetifs 
* de manipuler les éléments du réseau d'un cluster Kubernetes
* d'exposer un service fonctionnant dans un cluster au reste du monde

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

* On récupère le nom de notre service :
```
kubectl get svc
```
* Ce qui affiche :
```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx        ClusterIP   100.70.204.237   <none>        80/TCP    15m
```
* On supprime le service précédent:
```
kubectl delete svc nginx
```
* Puis on recréé un service de type NodePort:
```
kubectl expose deployment nginx --port 80 --type NodePort
```
* En affichant les services de nouveaux quelle différence remarquez vous ?:
```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx        NodePort    100.65.29.172  <none>        80:32593/TCP   8s
```
* Il n'y a toujours pas d'adresse IP externe, par contre le port 80 est mappé sur le port 32593, cela signifie que nous pouvons accéders à notre pod, via l'IP du node et le port 32593
* Pour obtenir l'IP du node :
```
kubectl get nodes -o wide
```
* On obtient un resultat comme celui-ci :
```
NAME       STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION      CONTAINER-RUNTIME
minikube   Ready    master   18h   v1.19.2   192.168.49.2   <none>        Ubuntu 20.04 LTS   4.4.111+   docker://19.3.8

```
* Il ne reste plus qu'à tester avec l'IP interne du Node concerné :
```
curl -s 192.168.49.2:32593 | grep h1
```
* Qui devrait afficher :
```
<h1>Welcome to nginx!</h1>
```
* Nous pouvons maintenant accéder au POD, via l'IP du Node, ce qui signifie qu'il est nécessaire de fournir l'adresse IP du Node, hors celle-ci peut changer et donc devoir être modifié. 

### Load Balancer 
Le service de type load balancer va permettre d'obtenir une adresse IP externe pour le cluster et ainsi d'accéder aux applications dans les pods, sans besoin de connaitre l'IP des nodes
* On commence par supprimer l'ancien service :
```
kubectl delete svc nginx
```
* Puis on recrée un service de type loadbalancer :
```
kubectl expose deployment nginx --port 80 --type LoadBalancer
```
* Après quelques minutes votre cluster obtient une adresse IP externe que vous pouvez voir avec :
```
kubectl get svc
```
* Exemple :
```
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx        LoadBalancer   100.69.15.89   35.205.60.29  80:31354/TCP   5s
```
* Il est maintenant possible d'accéder directement à notre application via l'IP du cluster :
```
curl -s 35.205.60.29 | grep h1
```
* Renvoit bien : 
```
<h1>Welcome to nginx!</h1>
```

-> Félicitations vous savez ajouter des services et manipuler les bases du réseau de K8s
* NB : sur Minikube en local, aucun système de load balancing n'est en place, vous n'obtiendrez donc pas d'adresse IP externe (publique), si vous voulez en obtenir une il faut :
* * soit utiliser un fournisseur de cloud qui inclut ce service 
* * soit configurer un outil de load balancing en local (il est possible d'utiliser minikube tunnel, qui permet d'accèder sur l'hote de Minikube, donc dans nos exercices sur la VM, à l'adresse IP externe : https://minikube.sigs.k8s.io/docs/handbook/accessing/ )
