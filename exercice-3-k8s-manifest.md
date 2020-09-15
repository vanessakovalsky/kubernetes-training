# Création du premier manifest

Cet exercice a pour objectif de :
* Rédiger un manifest.yaml qui contiendra les deux déploiements précédents et le service
* Lancer la creations des ressources à partir de ce manifest

## Création du manifest.yaml

* Créer un fichier manifest.yaml qui va contenir les différentes ressources à ajouter dans notre cluster 
* Ce fichier contient un premier deploiement qui contient le cluster nginx :
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx          # arbitrary label on deployment
spec:
  replicas: 1
  selector:
    matchLabels:        # labels the replica selector should match
      app: nginx
  template:
    metadata:
      labels:
        app: nginx      # label for replica selector to match
        version: 1.7.9  # arbitrary label we can match on elsewhere
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
 ```
 * Ajouter au sein du même fichier le deuxième déploiement des exercices précédents (pour rappel, les objets doivent être séparés avec --- )
 * Ajouter au sein du même fichier le service de type loadbalancer pour créer le service (la syntaxe est dans le support de cours, ou sur la doc de K8S : https://kubernetes.io/fr/docs/concepts/services-networking/service/#loadbalancer )
 * Vous pouvez générer le fichier à partir de la commande impérative avec :
 ```
 kubectl expose deployment nginx --port 80 --type NodePort --dry-run -o yaml > service-nodeport.yaml
```

 ## Execution de notre fichier de déploiement
 * Pour lancer le fichier
 ```
 kubectl create -f manifest.yaml
 ```
 * Vérifier les deploiements avec 
 ```
 kubectl get deployments
 ```
 * Vérifier les pods avec :
 ```
 kubectl get pods
 ```
 * Vérifier les services avec :
 ```
 kubectl get svc
 ```
 * -> Vous savez maintenant déployer plusieurs ressources à partirs d'un seul fichier déclaratif
