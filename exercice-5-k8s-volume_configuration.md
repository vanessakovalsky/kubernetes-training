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
$ kubectl create -f pvc-nginx.yaml
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
apiVersion: extensions/v1beta1
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
Now, you access the Nginx instance using curl. You should get a "403 Forbidden", because now the volume you mounted is empty.

> Hint. You learned about exposing deployments on a NodePort in the [service
> discovery](02-service-discovery-and-loadbalancing.md) exercise.
> 
> Hint 2. You can curl the nodeport from any device you have access to from the internet; Your machine, your cloud instance, or the multitool container in the cluster. You learned about the multitool and running command inside a container pod in the [service discovery](02-service-discovery-and-loadbalancing.md) exercise.

```shell
$ kubectl exec -it multitool-<ID> bash
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

Exit the multitool pod again if you are using that.

Create a file in the `html` dir inside the nginx pod and add some text in it:

```shell
$ kubectl exec -it nginx-deployment-6665c87fd8-cc8k9 -- bash

root@nginx-deployment-6665c87fd8-cc8k9:/# echo "<h1>Welcome to Nginx</h1>This is Nginx with html directory mounted as a volume from GCE Storage."  > /usr/share/nginx/html/index.html
root@nginx-deployment-6665c87fd8-cc8k9:/#
```

Exit the nginx pod again. From the multitool container, run curl again, you should see the web page:

```shell
$ kubectl exec -it multitool-69d6b7fc59-gbghn bash

bash-4.4#
 curl 10.0.96.7
<h1>Welcome to Nginx</h1>This is Nginx with html directory mounted as a volume from GCE Storage.
bash-4.4#
```

Kill the pod:

```shell
$ kubectl delete pod nginx-deployment-6665c87fd8-cc8k9
pod "nginx-deployment-6665c87fd8-cc8k9" deleted
```

Check if it is up (notice a new pod):

```shell
$ kubectl get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP          NODE
multitool-69d6b7fc59-gbghn          1/1       Running   0          10m       10.0.97.8   gke-dcn-cluster-2-default-pool-4955357e-txm7
nginx-deployment-6665c87fd8-nh7bs   1/1       Running   0          1m        10.0.96.8   gke-dcn-cluster-2-default-pool-4955357e-8rnp
```

Again, from multitool, curl the new nginx pod. You should see the page you created in the previous step:

```shell
$ kubectl exec -it multitool-69d6b7fc59-gbghn bash
bash-4.4# curl 10.0.96.8
<h1>Welcome to Nginx</h1>This is Nginx with html directory mounted as a volume from GCE Storage.
bash-4.4# 
```

## Ajout et utilisations de configmaps et de secrets dans un pod :

# Secrets and ConfigMaps

Secrets are a way to store things that you do not want floating around in your code.

It's things like passwords for databases, API keys and certificates.

Similarly, configmaps are for configuration, that doesn't really belong in code but needs to change. Examples include loadbalancer configurations, jenkins configuration and so forth.

We will look at both these in this coming exercise.

## Secrets as environment variables

Our [magnificent app](./secrets/secretapp.js) requires its API key and language. Rather than hardcode this sensitive information and commit it to git for all the world to see, we source these values from environment variables.

The first step to fixing it, would be to make our variables as environmental variables.

We have sourced the values in the code like this:

```shell
  const language = process.env.LANGUAGE;
  const API_KEY = process.env.API_KEY;
```

Because we are reading from the env variables we can specify some default in the Dockerfile.  We have used this:

```shell
FROM node:9.1.0-alpine
EXPOSE 3000
ENV LANGUAGE English
ENV API_KEY 123-456-789
COPY secretapp.js .
ENTRYPOINT node secretapp.js
```

This image is available as `praqma/secrets-demo`. We can run that in our Kubernetes cluster by using the [the deployment file](./secrets/deployment.yml). Notice the env values added in the bottom.

Run the deployment by writing:

```shell
$ kubectl apply -f secrets/deployment.yml
deployment.extensions/envtest created
```

Expose the deployment on a nodeport, so you can see the running container.

> You learned about exposing nodeports in the [service discovery](02-service-discovery-and-loadbalancing.md) exercise. And remember that the application is running on port `3000`

Despite the default value in the Dockerfile, it should now be overwritten by the deployment env values!

However we just moved it from being hardcoded in our app to being hardcoded in our deployment file.

## Secrets using the kubernetes secret resource

Let's move the API key to a (generic) secret:

```shell
$ kubectl create secret generic apikey --from-literal=API_KEY=oneringtorulethemall
secret/apikey created
```

Kubernetes supports different kinds of preconfigured secrets, but for now we'll deal with a generic one.

Similarly for the language into a configmap:

```shell
$ kubectl create configmap language --from-literal=LANGUAGE=Orcish
configmap/language created
```

Similarly to all other objects, you can run "get" on them.

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

> Try to investigate the secret by using the kubectl describe command:
> ```shell
> $ kubectl describe secret apikey
> ```
> Note that the actual value of API_KEY is not shown. To see the encoded value use:
> ```shell
> $ kubectl get secret apikey -o yaml
> ```

Last step is to change the Kubernetes deployment file to use the secrets.

Change:

```shell
        env:
        - name: LANGUAGE
          value: Polish
        - name: API_KEY
          value: 333-444-555
```

To:

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

After you have edited the `deployment.yml` file (or you can use the prepared one
`secrets/final.deployment.yml`), you need to apply the new edition of the file
by issuing: `kubectl apply -f deployment.yml` .

You should now see the variables being loaded from configmap and secret respectively.

Pods are not recreated automatically when secrets or configmaps change, i.e. to
hot swapping the values becomes a two step process:

```shell
$ kubectl create configmap language --from-literal=LANGUAGE=Elvish -o yaml --dry-run | kubectl replace -f -
configmap/language replaced
$ kubectl create secret generic apikey --from-literal=API_KEY=andinthedarknessbindthem -o yaml --dry-run | kubectl replace -f -
secret/apikey replaced
```

Then delete the pod (so it's recreated with the replaced configmap and secret) :

```shell
$ kubectl delete pod envtest-3380598928-kgj9d
pod "envtest-3380598928-kgj9d" deleted
```

Access it in a webbrowser again, to see the updated values.

## Clean up

```shell
$ kubectl delete deployment envtest
$ kubectl delete service envtest
$ kubectl delete configmap language
$ kubectl delete secret apikey
```
