# Gérer la configuration des pods

Cet exercice a pour objectifs :
* de définir et d'utiliser des configsmaps et des secrets 


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
* Pour lire les variables d'environnement il faut les définir dans le container via son image. C'est ce que l'on fait avec le Dockerfile suivant (l'image est construite ici : https://hub.docker.com/r/praqma/secrets-demo/tags ) qui injecte des variables d'environnements --> rien à faire de votre côté, il s'agit seulement d'information sur le fonctionnement des images dockers et de leur variables d'environnement
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
