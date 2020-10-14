# Lab 2. Modifier l'application

Dans l'exercice 1, nous avons installer l'application exemple guestbook en utilisant Helm et en voyant les avantages par rapport à l'utilisation de `kubectl`. Comment faire maintenant pour mettre à jour l'application qui tourne ?

Dans cet exercice nous allons voir comment mettre à jour une application en cours d'execution lorsque le chart a été modifié. Pour cela nous allons faire des modifications sur le graphique d'origine de `guestbook`en :
* Supprimant les esclaves Redis et en utilisant seulement la BDD en-mémoire
* Changer le type de `LoadBalancer` à`NodePort`.



## Mettre à jour l'application avec Helm

Dans cette section nous allons mettre à jour l'application `guestbook-demo` déployée dans l'exercice 1 en utilisant Helm.

Avant de commencer, voyons comment Helm simplifie le processus comparé à l'utilisation directe de Kubernetes. Helm fais usage d'un  [template language](https://helm.sh/docs/chart_template_guide/getting_started/) qui fournit une grande flexibilité et puissance aux auteurs de graphique, ce qui supprime la complexité des graphiques aux utilisateurs. Dans cet exemple guestbook, nous verrons les possibilités suivantes des templates :

* Values : un objet qui fournit accès aux valeurs passées au graphique. Un exemple est dans le `guestbook-service`, qui contient une ligne :`type: {{ .Values.service.type }}`. Cettel igne fournit la possibilité de définir le type de service durant la mise à jour ou l'installation.
* Control structures: Aussi appelé "actions" dans le langage des template, les structures de contôrles permette à l'auteur du template de contrôle le flux de génération du template. Un exemple est dans le  `redis-slave-service`, qui contient la ligne  `{{- if .Values.redis.slaveEnabled -}}`. Cette ligne nous permet d'activer ou de désactiver les maitres / esclaves de REDIS pendant la mise à jour ou l'installation.

Le fichier complet montré ci-dessous, montre comment les fichiers deviennent redondant quant le flag `slaveEnabled` est désactivé et aussi comment la valeur du port est définie. Il y a d'autres exemples des fonctionnalités de templates dans les autres fichiers du graphique.

```yaml
{{- if .Values.redis.slaveEnabled -}}
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
spec:
  ports:
  - port: {{ .Values.redis.port }}
    targetPort: redis-server
  selector:
    app: redis
    role: slave
{{- end }}
```

Assez parlé de théorie, il est temps de pratiquer !

1. Commençons par vérifier l'application déployée dans l'exercice 1. Cela peut être fait en vérifiant les releases de Helm : 

   ```console
   helm list -n helm-demo
   ```

   Noter que nous spécifions l'espace de nom. S'il n'est pas spécifié, il utilise l'espace de nom du contexte courant. Vous devriez avoir une sortie similaire à la suivante :

   ```console
   $ helm list -n helm-demo
   NAME           NAMESPACE REVISION  UPDATED                                 STATUS    CHART            APP VERSION
   guestbook-demo helm-demo 1         2020-02-24 18:08:02.017401264 +0000 UTC deployed  guestbook-0.2.0
   ```

   La commande `list`fournit la liste des graphiques déployés (release) en donnant les informations sur la version du graphique, son namespace, le numéro de mise à jour (revision) etc.

1. Nous savons que la release est présente depuis l'exercice 1, nous pouvons maintenant mettre à jour l'application:

    ```console
    $ cd helm101/charts

    $ helm upgrade guestbook-demo ./guestbook --set redis.slaveEnabled=false,service.type=NodePort --namespace helm-demo
    Release "guestbook-demo" has been upgraded. Happy Helming!
    ...
    ```

    Une mise à jour de Helm prend une release existante et met à jour en fonction des informations que vous fournissez. Vous devriez avoir une sortie similaire à la suivante :

    ```console
    $ helm upgrade guestbook-demo ./guestbook --set redis.slaveEnabled=false,service.type=NodePort --namespace helm-demo
    Release "guestbook-demo" has been upgraded. Happy Helming!
    NAME: guestbook-demo
    LAST DEPLOYED: Tue Feb 25 14:23:27 2020
    NAMESPACE: helm-demo
    STATUS: deployed
    REVISION: 2
    TEST SUITE: None
    NOTES:
    1. Get the application URL by running these commands:
      export NODE_PORT=$(kubectl get --namespace helm-demo -o jsonpath="{.spec.ports[0].nodePort}" services guestbook-demo)
      export NODE_IP=$(kubectl get nodes --namespace helm-demo -o jsonpath="{.items[0].status.addresses[0].address}")
      echo http://$NODE_IP:$NODE_PORT
    ```

    La commande `upgrade` met-à-jour l'application avec la version spécifiée du graphique, elle supprime les ressources `redis-slave`, et met à jour le `service.type` de l'application vers `NodePort`.

    Vérifiez les mises-à-jour en utilisant :  `kubectl get all --namespace helm-demo`:

    ```console
    $ kubectl get all --namespace helm-demo
    NAME                                  READY   STATUS    RESTARTS   AGE
    pod/guestbook-demo-6c9cf8b9-dhqk9     1/1     Running   0          20h
    pod/guestbook-demo-6c9cf8b9-zddn2     1/1     Running   0          20h
    pod/redis-master-5d8b66464f-g7sh6     1/1     Running   0          20h

    NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    service/guestbook-demo   NodePort    172.21.43.244    <none>        3000:31202/TCP   20h
    service/redis-master     ClusterIP   172.21.12.43     <none>        6379/TCP         20h

    NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/guestbook-demo   2/2     2            2           20h
    deployment.apps/redis-master     1/1     1            1           20h

    NAME                                        DESIRED   CURRENT   READY   AGE
    replicaset.apps/guestbook-demo-6c9cf8b9     2         2         2       20h
    replicaset.apps/redis-master-5d8b66464f     1         1         1       20h

    ```

    > Note: Le type de service à changer (vers `NodePort`) et un nouveau port a été alloué(`31202` dans la sortie de notre commande) au service de guestbook. Toutes les ressources `redis-slave` ont été supprimées.

    Lorsque vous vérifier la release de helm avec `helm list -n helm-demo`, vous verrez la révision et la date ont été mise à jour.:

    ```console
    $ helm list -n helm-demo
    NAME            NAMESPACE REVISION  UPDATED                                 STATUS    CHART            APP VERSION
    guestbook-demo  helm-demo 2         2020-02-25 14:23:27.06732381 +0000 UTC  deployed  guestbook-0.2.0
    ```

1. Voir le guestbook

   Obtenir l'adresse IP publique d'un de nos noeuds : :
   ```console
   kubectl get nodes -o wide
   ```

   Naviguer à l'adresse IP  + le port du noeud qui s'affichait précédemment.

## Conclusion

Féliciations, vous savez mettre à jour vos application! Helm ne nécessite pas de changement manuel des ressources et est pourtant plus facile à mettre à jour! Toutes les configurations peuvent être définies à la volée en ligen de commande ou en utilisant des fichiers de surcharge. Cela est rendu possible en ajoutant la logique au fichiers de templates, qui active ou désactive les capactiés, en fonction des flags définis.

