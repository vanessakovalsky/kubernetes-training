
## D'ajouter des vérifications de l'état de vos containers (healthchecks)

* Lorsqu'un conteneur tourne dans un pod, Kubernetes vous donne 4 informations :
```
NAME                                       READY     STATUS    RESTARTS   AGE
mypod-somenumbers-guid                     1/1       Running   0          3h
```
* Intéressons nous à la partie Ready, qui est la vérification interne de fonctionnement que Kubernetes fait sur un conteneur.
* Lorsqu'un conteneur n'est pas en état Kubernetes n'envoit pas de trafic sur celui-ci.
* Parfois, certains conteneurs semble fonctionner alors qu'ils ont des problèmes. C'est pourquoi il est important de personnaliser les véfifications effectuées. 
* Par exemple dans le déploiement suivant, le pod va fonctionner correctement pendant 30 secondes, puis ne plus fonctionner. La vérification personnalisé permet alors de savoir qu'il ne fonctionne plus.
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
* La magie de Kubernetes consiste a recréé alors le conteneur qui ne fonctionne plus comme prévu. 

* Déployer ce déploiement avec  :kubectl apply -f <fichier-deploiement-a-enregistrer.yml>
* Vérifier les logs pour connaitre les critères de vérifications sur les autres pods :
```
kubectl describe pod liveness-exec
```  



<details>
    <summary>.La partie qui suit (dans le spoiler) est facultative, il s'agit de builder une image docker à utiliser dans K8s </summary> 


Créer une nouvelle image du conteneur (avec docker) et pusher la sur le hub
 * Connectez vous au conteneur et créer un fichier healthz.html dans /usr/share/nginx/ 

 * Créer un fichier healthz.html qui contient
```
<html>
<body>
Ma page de vérification que mon conteneur et mon pod fonctionne
</body>
</html>
```
  * Créer un fichier Dockerfile qui contient 
```
FROM nginx
COPY healthz.html /usr/share/nginx/html
```
  * Builder l'image : 
```
docker build -t myhealthcheck . 
```
  * Ajouter un tag à votre image et commiter là, et envoyer la sur le serveur :
```
docker run -d --name mycheck myhealthcheck
docker commit mycheck vanessakovalsky/my-healtcheck:v1
docker push vanessakovalsky/my-healtcheck:v1
```
</details>


* Créer un déploiement avec votre nouvelle image (ou celle-ci si vous n'avez pas builder: vanessakovalsky/my-healtcheck:v1 /!\ a mettre le tag ) ajouter ces indicateurs en plus du reste dans le manifest de deploiement pour spécifier une personnalisation des vérifications
```
  livenessProbe:
      httpGet:
        path: /healthz.html #Your endpoint for health check.
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3
```
* Vérifier que le healtch continue à fonctionner;. (vous pouvez pas exemple renommer le fichier pour vérifier sur le conteneur lorsqu'il est en cours d'execution 
