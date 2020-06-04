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
* Vérifier le nombre de pods sur le déploiement :

```shell
$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
multitool   1/1     1            1           24m
nginx       4/4     4            4           34m
```

```shell
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          24m
nginx-569477d6d8-4msf8       1/1       Running   0          20m
nginx-569477d6d8-bv77k       1/1       Running   0          34s
nginx-569477d6d8-s6lsn       1/1       Running   0          34s
nginx-569477d6d8-v8srx       1/1       Running   0          35s
```

**Notice:** The nginx deployment says Ready=4/4, Up-to-date=4, Available=4. And the pods also show the same. There are now 4 nginx pods running; one of them was already running (being older), and the other three are started just now.

You can also scale down! - e.g. to 2:

```shell
$ kubectl scale deployment nginx --replicas=2
deployment.extensions/nginx scaled
```

```shell
$ kubectl get pods
NAME                         READY     STATUS        RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running       0          25m
nginx-569477d6d8-4msf8       1/1       Running       0          21m
nginx-569477d6d8-bv77k       0/1       Terminating   0          1m
nginx-569477d6d8-s6lsn       0/1       Terminating   0          1m
nginx-569477d6d8-v8srx       1/1       Running       0          2m
```

Notice that unnecessary pods are killed immediately.

```shell
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          26m
nginx-569477d6d8-4msf8       1/1       Running   0          22m
nginx-569477d6d8-v8srx       1/1       Running   0          2m
```

You can delete the nginx deployment and service at this point. We have no use for these anymore. Besides, you can always re-create them.

```shell
$ kubectl delete deployment nginx
deployment.extensions "nginx" deleted
$ kubectl delete service nginx
service "nginx" deleted
```

## Extra-credit: High Availability Exercise

To prove that multiple pods of the same deployment provide high availability, we do a small exercise. To visualize it, we need to run a small web server which could return us some unique content when we access it. We will use our trusted multitool for it. Let's run it as a separate deployment and access it from our local computer.

```shell
$ kubectl create deployment customnginx --image=praqma/network-multitool
deployment.apps/customnginx created
$ kubectl scale deployment customnginx --replicas=4
deployment.extensions/customnginx scaled
```

```shell
$ kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
customnginx-3557040084-1z489   1/1       Running   0          49s
customnginx-3557040084-3hhlt   1/1       Running   0          49s
customnginx-3557040084-c6skw   1/1       Running   0          49s
customnginx-3557040084-fw1t3   1/1       Running   0          49s
multitool-5f9bdcb789-k7f4q     1/1       Running   0          19m
```

Let's create a service for this deployment as a type=LoadBalancer:

```shell
$ kubectl expose deployment customnginx --port=80 --type=LoadBalancer
service/customnginx exposed
```

Verify the service and note the public IP address:

```shell
$ kubectl get services
NAME          TYPE           CLUSTER-IP    EXTERNAL-IP        PORT(S)        AGE
customnginx   LoadBalancer   100.67.40.4   35.205.60.41       80:30087/TCP   1m
```

Query the service, so we know it works as expected:

```shell
$ curl -s 35.205.60.41
Praqma Network MultiTool (with NGINX) - customnginx-7cf9899b84-rjgrb - 10.8.2.47/24
```

Next, setup a small bash loop on your local computer to curl this IP address, and get it's IP address.

```shell
$ while true; do sleep 1; curl -s 35.205.60.41; done
Praqma Network MultiTool (with NGINX) - customnginx-7fcfd947cf-zbvtd - 100.96.2.36 <BR></p>
Praqma Network MultiTool (with NGINX) - customnginx-7fcfd947cf-zbvtd - 100.96.1.150 <BR></p>
Praqma Network MultiTool (with NGINX) - customnginx-7fcfd947cf-zbvtd - 100.96.2.37 <BR></p>
Praqma Network MultiTool (with NGINX) - customnginx-7fcfd947cf-zbvtd - 100.96.2.37 <BR></p>
Praqma Network MultiTool (with NGINX) - customnginx-7fcfd947cf-zbvtd - 100.96.2.36 <BR></p>
^C
```

We see that when we query the LoadBalancer IP, it is giving us result/content from all four containers. None of the curl commands is timed out. Now, if we kill three out of four pods, the service should still respond, without timing out. We let the loop run in a separate terminal, and kill three pods of this deployment from another terminal.

```shell
$ kubectl delete pod customnginx-3557040084-1z489 customnginx-3557040084-3hhlt customnginx-3557040084-c6skw
pod "customnginx-3557040084-1z489" deleted
pod "customnginx-3557040084-3hhlt" deleted
pod "customnginx-3557040084-c6skw" deleted
```

Immediately check the other terminal for any failed curl commands or timeouts.

```shell
Container IP: 100.96.1.150 <BR></p>
Container IP: 100.96.1.150 <BR></p>
Container IP: 100.96.2.37 <BR></p>
Container IP: 100.96.1.149 <BR></p>
Container IP: 100.96.1.149 <BR></p>
Container IP: 100.96.1.150 <BR></p>
Container IP: 100.96.2.36 <BR></p>
Container IP: 100.96.2.37 <BR></p>
Container IP: 100.96.2.37 <BR></p>
Container IP: 100.96.2.38 <BR></p>
Container IP: 100.96.2.38 <BR></p>
Container IP: 100.96.2.38 <BR></p>
Container IP: 100.96.1.151 <BR></p>
```

We notice that no curl command failed, and actually we have started seeing new IPs. Why is that? It is because, as soon as the pods are deleted, the deployment sees that it's desired state is four pods, and there is only one running, so it immediately starts three more to reach that desired state. And, while the pods are in process of starting, one surviving pod takes the traffic.

```shell
$ kubectl get pods
NAME                           READY     STATUS        RESTARTS   AGE
customnginx-3557040084-0s7l8   1/1       Running       0          15s
customnginx-3557040084-1z489   1/1       Terminating   0          16m
customnginx-3557040084-3hhlt   1/1       Terminating   0          16m
customnginx-3557040084-bvtnh   1/1       Running       0          15s
customnginx-3557040084-c6skw   1/1       Terminating   0          16m
customnginx-3557040084-fw1t3   1/1       Running       0          16m
customnginx-3557040084-xqk1n   1/1       Running       0          15s
```

This proves, Kubernetes provides us High Availability, using multiple replicas of a pod.

## Clean up

Delete deployments and services as follows:

```shell
$ kubectl delete deployment customnginx
$ kubectl delete deployment multitool
$ kubectl delete service customnginx
```


## De faire des mises à jours de déploiements
Recreate the nginx deployment that we did earlier:

```shell
$ kubectl create deployment nginx --image=nginx:1.7.9
```

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
