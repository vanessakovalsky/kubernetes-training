# Deploiement de pods, mises à l'échelle et vérification

Cet exercice a pour objectfs :

* De vous permettre de mettre à l'échelle vos pod dans votre cluster
* De faire des mises à jours de déploiements
* D'ajouter des vérifications de l'état de vos containers (healthchecks)

## De vous permettre de mettre à l'échelle vos pod dans votre cluster
Nous allons travailler sur la scalabilité de notre pod, afin de permettre d'augmenter le nombre de replicas disponibles.$

* Commencer par augmenter le nombre de replicas disponible à 4 pour notre déploiement nginx :
```shell
kubectl scale deployment nginx --replicas=4
deployment.extensions/nginx scaled
```
* Vérifier le résultat sur les déploiements et les pods :

```shell
kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
multitool   1/1     1            1           24m
nginx       4/4     4            4           34m
```

```shell
kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          24m
nginx-569477d6d8-4msf8       1/1       Running   0          20m
nginx-569477d6d8-bv77k       1/1       Running   0          34s
nginx-569477d6d8-s6lsn       1/1       Running   0          34s
nginx-569477d6d8-v8srx       1/1       Running   0          35s
```

**NB:** Le déploiement nginx dit Ready=4/4, Up-to-date=4, Available=4. Les pods montre la même chose, il y a maintenant 4 pods nginx qui fonctionnenet. L'un d'eux est le premier a avoir été lancer, et les autres ont été créé quelques secondes auparavant par notre commande précédente.

* Il est également possible de réduire l'échelle, par exemple à 2 :

```shell
$ kubectl scale deployment nginx --replicas=2
deployment.extensions/nginx scaled
```
* On vérifie le résultat :
```shell
kubectl get pods
NAME                         READY     STATUS        RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running       0          25m
nginx-569477d6d8-4msf8       1/1       Running       0          21m
nginx-569477d6d8-bv77k       0/1       Terminating   0          1m
nginx-569477d6d8-s6lsn       0/1       Terminating   0          1m
nginx-569477d6d8-v8srx       1/1       Running       0          2m
```
* Les pods nont utilisé sont tués immédiatement :

```shell
kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          26m
nginx-569477d6d8-4msf8       1/1       Running   0          22m
nginx-569477d6d8-v8srx       1/1       Running   0          2m
```

## De faire des mises à jours de déploiements

And expose the pod using a load balancer service (remember that it might take a
few minutes for the cloud infrastructure to deploy the load balancer, i.e. the
external IP might be shown as `pending`):

```shell
$ kubectl expose deployment nginx --port 80 --type LoadBalancer
```

Note down the loadbalancer IP from the services command:

```shell
$ kubectl get service
```

Increase the replicas to four:

```shell
$ kubectl scale deployment nginx --replicas=4
```

From another terminal on your machine check (using load balancer IP) which version is currently running and to see changes when rollout is happening:

```shell
$ while true; do  curl -sI 35.205.60.29  | grep Server; sleep 2; done
```

## Update Deployment

Rollout an update to the image:

```shell
$ kubectl set image deployment nginx nginx=nginx:1.9.1 --record
```

Check the rollout status:

```shell
$ kubectl rollout status deployment nginx
```

Investigate rollout history:

```shell
$ kubectl rollout history deployment nginx
```

Try rolling out other image version by repeating the `set image` command from
above.  Suggested image versions are 1.12.2, 1.13.12, 1.14.1, 1.15.2.

Try also rolling out a version that does not exist:

```shell
$ kubectl set image deployment nginx nginx=nginx:100.200.300 --record
```

what happened - do the curl operation still work?  Investigate the running pods with:

```shell
$ kubectl get pods
```

## Undo Update

The rollout above using a non-existing image version caused some pods to be
non-functioning. Next, we will undo this faulty deployment. First, investigate
rollout history:

```shell
$ kubectl rollout history deployment nginx
```

Undo the rollout and restore the previous version:

```shell
$ kubectl rollout undo deployment nginx
```

Investigate the running pods:

```shell
$ kubectl get pods
```

## Clean up

Delete deployments and services as follow:

```shell
$ kubectl delete deployment nginx
$ kubectl delete service nginx
```
## D'ajouter des vérifications de l'état de vos containers (healthchecks)
Working with Kubernetes, you eventually need to understand the "magic". 

When a container runs in a pod, Kubernetes reports back four things: 
```
NAME                                       READY     STATUS    RESTARTS   AGE
mypod-somenumbers-guid                     1/1       Running   0          3h
```

This workshop assignment looks at the ready part, which is the internal health check Kubernetes performs on a container. 

The difference between a container being healthy or unhealthy, is vital. A container can be creating, failing or otherwise deployed but unavailable - and in this state Kubernetes will choose not to route traffic to the container if it deems it unhealthy. 

However, in some cases an app looks "healthy" despite having issues. This is where customized health checks become important. 

Examples include the database running but being unreachable, the app functioning but a volume to store files in being unavailable and so on. 

[This deployment](health-checks/deployment.yml) shows this quite nicely. 

For the first 30 seconds of the pod's lifespan, it will be healthy. After this, the custom health check will fail. 

The magic in Kubernetes, is that it will recreate an unhealthy container. 

First, apply the deployment file with the `kubectl apply -f` command. Remember to specify the path.

Look at the logs: 
```
kubectl describe pod liveness-exec
```  

Any (http) code greater than or equal to 200 and less than 400 indicates success. Any other code indicates failure.

Let's go back to our applications, and create a custom health check. 

Create an endpoint that does something customized, and returns 200. (Make sure it can fail and return an error above 500 if you want to test). 

Push the container, and create a deployment for it and include: 

```
  livenessProbe:
      httpGet:
        path: /healthz #Your endpoint for health check.
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

Verify that the healthcheck fails when conditions are not met. 

This concludes the exercise for health checks!
