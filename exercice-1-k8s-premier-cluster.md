# Créer le premier cluster

Cet exercice a pour objectif de : 
* Installer les outils nécessaires à l'utilisation de K8S
* Créer un premier cluster

## Installation des outils 

Nous allons avoir besoin de différents outils pour faire fonctionner Kubernetes sur notre poste

* kubectl, pour l'interface ligne de commande (CLI) : https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/ 
* un moteur de conteneur de type docker : https://docs.docker.com/get-docker/ ou un hyperviseur de type virtualbox : https://www.virtualbox.org/wiki/Downloads 
* Minkube pour lancer et manipuler les clusters : https://kubernetes.io/fr/docs/tasks/tools/install-minikube/
* Pour vérifier que cela fonctionne : 
```
minikube start --driver=docker
```
* La console va alors vous renvoyer un affichage de type : 
```
Starting local Kubernetes cluster...
Running pre-create checks...
Creating machine...
Starting local Kubernetes cluster...
```
* Pour vérifier l'état du cluster :
```
minikube status
```
* Cette commande affiche :
```
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```
* Pour arrêter le cluster (une fois l'exercice complètement terminé) :
```
minikube stop
```

## Mise en place de nos premiers pods 
* Pour commencer creer un namespace pour isoler notre travail :
```
kubectl create namespace my-namespace
```
* Puis définir notre namespace comme celui par défaut utilisé par k8s
```
kubectl config set-context $(kubectl config current-context) --namespace=my-namespace
```
* Pour vérifier on affiche le contexte : 
```
kubectl config get-contexts
```
* Utiliser la commande create de kubectl pour déployer votre premier pod :
```
kubectl create deployment multitool --image=praqma/network-multitool
deployment.apps/multitool created
```
* Vérifier si le déploiement a bien été fait 
```
kubectl get deployments
```
* Voir les pods créés lors du deploiement :
```
kubectl get pods
```
* Cette commande vous renvoit :
```
NAME                            READY     STATUS    RESTARTS   AGE
po/multitool-3148954972-k8q06   1/1       Running   0          25m
```
* Pour voir les logs d'un pod :
```
kubectl logs po/multitool-3148954972-k8q06
```
* Supprimer un pod :
```
kubectl delete pod po/multitool-3148954972-k8q06
```
* Lancer immédiatement un kubectl get pods, qu'observez vous ? 
* Pour supprimer un déploiement :
```
kubectl delete deployment multitool
```

-> Félicitations vous avez installer l'environnement nécessaire pour kubernetes et manipuler des objets basiques
