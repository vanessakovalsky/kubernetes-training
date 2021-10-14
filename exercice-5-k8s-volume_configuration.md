# Ajouter un volume, gérer la configuration des pods

Cet exercice a pour objectifs :
* d'ajouter un volume stockage et une requete vers ce volume de stockage depuis notre pod nginx
* de définir et d'utiliser des configsmaps et des secrets 

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
        image: nginx:1.9.1
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

## Ajout et utilisations de configmaps et de secrets dans un pod :
Nous allons a présents travailler avec une application JS qui a besoin d'une clé d'API et d'une langue.
Plutôt que de coder en dur ces informations sensibles, nous allons utiliser les secrets et configmaps mis à disposition par kubernetes.

L'application est ici : [magnificent app](./secrets/secretapp.js)

###  Secrets comme variables d'environnement 

*La première étape est d'utiliser les variables d'environnement en tant que variable.
Dans le code source cela est représenté par :

```shell
  const language = process.env.LANGUAGE;
  const API_KEY = process.env.API_KEY;
```
* Pour lire les variables d'environnement il faut les définir dans le container via son image. C'est ce que l'on fait avec le Dockerfil suivant (à créer du coup)
```shell
FROM node:9.1.0-alpine
EXPOSE 3000
ENV LANGUAGE English
ENV API_KEY 123-456-789
COPY secretapp.js .
ENTRYPOINT node secretapp.js
```
* Puis nous créons un fichier de déploiement deployment.yml avec le contenu 
```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envtest
spec:
  replicas: 1
  selector:
    matchLabels:        # labels the replica selector should match
      name: envtest
  template:
    metadata:
      labels:
        name: envtest
    spec:       
      containers:
        - name: envtest
          image: praqma/secrets-demo
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: LANGUAGE
              value: Polish
            - name: API_KEY
              value: 333-444-555

```
* Lancer le déploiement avec la commande :
```shell
kubectl apply -f deployment.yml
deployment.extensions/envtest created
```
* Exposer le déploiement avec node port pour vérifier que l'application se lance et utiliser curl sur l'adresse ip du cluster avec le port donnée par node port pour accéder au port 3000 de votre container
Expose the deployment on a nodeport, so you can see the running container.

**NB** Les valeurs par défaut présentes dans le dockerfile sont maintenant surchargé avec celles contenus dans le déploiement, cela permet de définir des valeurs différentes pour les différents environnements par exemple.

## Secrets using the kubernetes secret resource
L'étape précédente nous a permis de rendre variables certaines informations, cependant définir une clé d'API en clair dans un fichier, n'est pas toujours sécurisé. Utilions maintenant l'objet secret de Kubernetes pour passer ces informations de manières sécurisées à nos déploiement
* Générer un secret pour notre clé d'API :
```shell
kubectl create secret generic apikey --from-literal=API_KEY=oneringtorulethemall
secret/apikey created
```
* Kubernetes supports différents type de secrets préconfigurés, nous utilions ici le type générique. Pour obtenir la liste des types de secrets disponibles, vous pouvez utiliser :
```shell
kubectl create secret --help
```
* Pour le langage, nous créons cette fois-ci un configmap :
```shell
$ kubectl create configmap language --from-literal=LANGUAGE=Orcish
configmap/language created
```
* Pour obtenir la liste des secrets, on peut utiliser get sur l'objet secret :
```shell
$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
apikey                Opaque                                1         4m
default-token-td78d   kubernetes.io/service-account-token   3         3h
```

```shell
$ kubectl get configmaps
NAME       DATA      AGE
language   1         2m
```
* Vous pouvez essayer d'afficher le secret avec describe :
> ```shell
> $ kubectl describe secret apikey
> ```
* **NB** La valeur du secret n'est pas montré en clair, on voit la valeur chiffré.
* Dernière étape, modifier le fichier de déploiement pour utiliser les secrets creer:
Remplacer:

```shell
        env:
        - name: LANGUAGE
          value: Polish
        - name: API_KEY
          value: 333-444-555
```
Par : 
```shell
        env:
        - name: LANGUAGE
          valueFrom:
            configMapKeyRef:
              name: language
              key: LANGUAGE
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: apikey
              key: API_KEY
```
* Relancer le déploiement avec
```
kubectl apply -f deployment.yml
```
* Les variables sont maintenant chargées depuis configmap et secret
* Les pods ne sont pas recréés automatiquement lors des modification des secrets ou des configmaps.
* Il est nécessaire de les remplacer en deux étapes :
```shell
kubectl create configmap language --from-literal=LANGUAGE=Elvish -o yaml --dry-run | kubectl replace -f -
configmap/language replaced
kubectl create secret generic apikey --from-literal=API_KEY=andinthedarknessbindthem -o yaml --dry-run | kubectl replace -f -
secret/apikey replaced
```
**NB** L'option --dry-run permet de simuler l'execution sans la lancer réellement.
* Puis il faut supprimer le pod, pour que K8S le recrée avec les bons configmaps et secrets :
```shell
kubectl delete pod envtest-3380598928-kgj9d
pod "envtest-3380598928-kgj9d" deleted
```
* Accéder à l'application via Curl pour voir les valeurs mises à jour
