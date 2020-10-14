# Exercice 3. Garder la trace de l'application déployée

Partons du principe que vous avez déployé plusieurs versions de votre application (c'est-à-dire que vous avez mis à jour l'application en cours d'éxecution). Comment garder une trace des versions et faire un retour arrière ?

## Gestion des révisions avec Helm

Dans cet exercice, nous illustrons la gestion des révisions sur l'application déployée `guestbook-demo` en utilisant Helm.

Avec Helm, à chaque installation, mise-)-jour ou retour arrière est fait, le numéro de révision est incrémenté de 1. Le premier numéro de révision est toujours 1. Helm enregistre les métadonnées des relases dans les Secrets ou les ConfigMaps, stocké sur le cluster Kubernetes. Chaque fois que votre release change, cela s'ajoute aux données existantes. Cela permet à Helm de revenir à une précédente release.

Voyons en pratique comment cela fonctionne.

1. Vérifier le nombre de déploiements :

    ```console
    helm history guestbook-demo -n helm-demo
    ```

    Vous devriez avoir un retour similaire au suivant puisque dans l'exercice 2 nous avons faite une mise à jour, après l'installation initiale de l'exercice 1.

    ```console
    $ helm history guestbook-demo -n helm-demo
    REVISION    UPDATED                     STATUS      CHART           APP VERSION DESCRIPTION
    1           Mon Feb 24 18:08:02 2020    superseded  guestbook-0.2.0             Install complete
    2           Tue Feb 25 14:23:27 2020    deployed    guestbook-0.2.0             Upgrade complete
    ```

1. Revenir à une révision précédente :

    Lors d'un retour en arrière, Helm vérifie les changements qui ont été fait lors de la mise à jour entre la révision 1 et la révision 2. Cette information permet de créer les appels au serveur d'API de Kubernetes, pour mettre à jour l'application déployée et revenir à l'état initial du déploiement - autrement dit, avec les esclaves Redis et l'utilisation du load balancer..

    Revenir en arrière avec la commande :

    ```console
    helm rollback guestbook-demo 1 -n helm-demo
    ```

    ```console
    $ helm rollback guestbook-demo 1 -n helm-demo
    Rollback was a success! Happy Helming!
    ```

    Vérifier à nouveau l'historique :

    ```console
    helm history guestbook-demo -n helm-demo
    ```

    Vous devriez avoir un retour similaire au suivant:

    ```console
    $ helm history guestbook-demo -n helm-demo
    REVISION    UPDATED                     STATUS      CHART           APP VERSION DESCRIPTION
    1           Mon Feb 24 18:08:02 2020    superseded  guestbook-0.2.0             Install complete
    2           Tue Feb 25 14:23:27 2020    superseded  guestbook-0.2.0             Upgrade complete
    3           Tue Feb 25 14:53:45 2020    deployed    guestbook-0.2.0             Rollback to 1
    ```

    Vérifier le retour en arrière, en utilisant :

    ```console
    kubectl get all --namespace helm-demo
    ```

    ```console
    $ kubectl get all --namespace helm-demo
    NAME                                  READY   STATUS    RESTARTS   AGE
    pod/guestbook-demo-6c9cf8b9-dhqk9     1/1     Running   0          20h
    pod/guestbook-demo-6c9cf8b9-zddn      1/1     Running   0          20h
    pod/redis-master-5d8b66464f-g7sh6     1/1     Running   0          20h
    pod/redis-slave-586b4c847c-tkfj5      1/1     Running   0          5m15s
    pod/redis-slave-586b4c847c-xxrdn      1/1     Running   0          5m15s

    NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    service/guestbook-demo   LoadBalancer   172.21.43.244    <pending>     3000:31202/TCP   20h
    service/redis-master     ClusterIP      172.21.12.43     <none>        6379/TCP         20h
    service/redis-slave      ClusterIP      172.21.232.16    <none>        6379/TCP         5m15s

    NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/guestbook-demo   2/2     2            2           20h
    deployment.apps/redis-master     1/1     1            1           20h
    deployment.apps/redis-slave      2/2     2            2           5m15s

    NAME                                        DESIRED   CURRENT   READY   AGE
    replicaset.apps/guestbook-demo-26c9cf8b9    2         2         2       20h
    replicaset.apps/redis-master-5d8b66464f     1         1         1       20h
    replicaset.apps/redis-slave-586b4c847c      2         2         2       5m15s
    ```

    Vous pouvez maintenant voir sur la sortie, que le service de l'app est de type `LoadBalancer` de nouveau et que le déploiement des maitre/esclaves de Redis sont revenus.
    Cela montre un complet retour en arrière suite à la mise  jour de l'exercice 2.

## Conclusion

Avec cette exercice, nous pouvons dire que Helm gère bien les révisions, alors que Kubernetes ne le permet pas nativement! Vous devez vous demandez pourquoi avons nous besoin de faire `helm rollback` alors que nous pourrions relancer la commande  `helm upgrade` depuis une ancienne version. C'est une bonne question. Techniquement, vous obtiendrez les mêmes ressources (avec les mêmes paramètres) deployées. Cependant, l'avantage de l'utilisation de  `helm rollback` est que helm gère (au sens se souvient) toutes les variations / paramètres des précédents `helm install\upgrade` pour vous. Faire le retour arrière en utilisant `helm upgrade` nécessite que vous et votre équipe au complet de tracer manuellement comment les commandes ont été précédemment éxecutées. C'est non seulement pénible, mais surtout sujet à erreur. Il est plus facile, sécurisé et réutilisable de laisser Helm gérer cela pour vous et de lui dire seulement à quelle version antérieur revenir, et Helm s'occupe du reste.

