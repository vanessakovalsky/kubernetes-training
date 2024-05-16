# Ajouter un volume et l'associer à des pods

Cet exercice a pour objectifs :
* d'ajouter un volume stockage et une requete vers ce volume de stockage depuis notre pod nginx

## Ajout et utilisation de volume dans un pod :
* Créer un fichier pvc-nginx.yml avec le contenu suivant :
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nginx
  labels:
    name: pvc-nginx
    app: efk
spec:
  # If you are using dynamic provisioning, it is important to specify a storageClassName.
  storageClassName: "standard"
  #selector:
   # matchLabels:
    #  name: pv-nginx
  accessModes:
    # Though accessmode is already defined in pv definition. It is still needed here.
    # - ReadWriteMany
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
* Créer une requete pour un volume provisionné dynamiquemenet pour nginx à partir du fichier  :

```shell
$ kubectl create -f pvc-nginx.yml
```
* Vérifier que le PVC existe et est fonctionnel :
```shell
kubectl get pvc
```
* Exemple:
```shell
kubectl get pvc
NAME        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nginx   Bound     pvc-e8a4fc89-2bae-11e8-b065-42010a8400e3   5Gi        RWO            standard       4m
```
* Il devrait y avoir un volume persistent auto-crée avec la PVC

```shell
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM               STORAGECLASS   REASON    AGE
pvc-e8a4fc89-2bae-11e8-b065-42010a8400e3   5Gi        RWO            Delete           Bound     default/pvc-nginx   standard                 5m
```
* Maintenant nous allons créer et attacher un PV et un PVC à notre déploiement nginx
* Créer un fichier/nginx-persistent-storage.yaml contenant:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
      - name: nginx-htmldir-volume
        persistentVolumeClaim:
          claimName: pvc-nginx
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 443
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: nginx-htmldir-volume
```
* Redéployer nginx

```shell
kubectl create -f nginx-persistent-storage.yaml
```
* Après le démarrage, vous pouvez vérifier ce qui a été déployer avec : 
```shell
kubectl describe pod nginx
```
* Si vous acceceder avec curl (en exposant un service comme vu précédemment) au conteneur, vous obtenez une erreur 403, c'est normal le volume monté est vide :
```shell
kubectl exec -it multitool-<ID> bash
bash-4.4# curl 10.0.96.7
<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.9.1</center>
</body>
</html>
bash-4.4# 
```
* Sortez du pod avec exit
* Créer un fichier HTML dans le répertoire utilisé par le pod nginx et ajouté du contenu :
```shell
kubectl exec -it nginx-deployment-6665c87fd8-cc8k9 -- bash
root@nginx-deployment-6665c87fd8-cc8k9:/# echo "<h1>Welcome to Nginx</h1>This is Nginx with html directory mounted as a volume from GCE Storage."  > /usr/share/nginx/html/index.html
root@nginx-deployment-6665c87fd8-cc8k9:/#
```
* Sortir du pod avec exit. et lancer curl de nouveau pour voir votre page web :
```shell
kubectl exec -it multitool-69d6b7fc59-gbghn bash

bash-4.4#
 curl 10.0.96.7
<h1>Welcome to Nginx</h1>This is Nginx with html directory mounted as a volume from GCE Storage.
bash-4.4#
```
* Vous savez maintenant créer, monter et utiliser un volume et une requête sur ce volume. 

