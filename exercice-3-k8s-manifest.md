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
        version: latest  # arbitrary label we can match on elsewhere
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
 ```
 * Ajouter au sein du même fichier le deuxième déploiement des exercices précédents (multitool) (pour rappel, les objets doivent être séparés avec --- )


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
 * -> Vous savez maintenant déployer plusieurs ressources à partirs d'un seul fichier déclaratif
