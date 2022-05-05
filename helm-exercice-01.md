# Exercice 1 - Premier déploiement

Nous allons voir comment Helm peut nous aider à rester concentrer en suivant un graphique qui fait le travail pour nous. Commençons avec le déploiement d'une application sur un cluster Kuberetes en utilisant `kubectl` puis nous verrons comment alléger notre travail en déployant la même application avec helm.

L'application est [Guestbook App](https://github.com/IBM/guestbook),qui est une application web multi-tier d'exemple.

## Pré-requis

Avoir un environnement Kubernetes installé et fonctionnel (voir exercice 1 - Installation) .
Si vous ne pouvez pas installer en local, vous pouvez utiliser le bac à sable de KataCoda : https://www.katacoda.com/courses/kubernetes/playground

## Scenario 1: Deployiement de l'application en utilisant `kubectl`

Dans cette partie de l'exercice, nous allons déployer l'application en utilisant le client Kubernetes `kubectl`. Nous utilisons la [Version 1](https://github.com/IBM/guestbook/tree/master/v1) de l'application pour ce déploiement :

Cloner le dépôt [Guestbook App](https://github.com/IBM/guestbook) pour obtenir les fichiers:

```console
git clone https://github.com/IBM/guestbook.git
```

1. Utiliser les fichiers de configurations du dépôt git cloné pour déployer les conteneurs et créers les services en utilisant les commandes suivantes ::

   ```console
   $ cd guestbook/v1

   $ kubectl create -f redis-master-deployment.yaml
   deployment.apps/redis-master created

   $ kubectl create -f redis-master-service.yaml
   service/redis-master created

   $ kubectl create -f redis-slave-deployment.yaml
   deployment.apps/redis-slave created

   $ kubectl create -f redis-slave-service.yaml
   service/redis-slave created

   $ kubectl create -f guestbook-deployment.yaml
   deployment.apps/guestbook-v1 created

   $ kubectl create -f guestbook-service.yaml
   service/guestbook created
   ```


1. Voir l'application:

   Vous pouvez maintenant jouer avec l'application guestbook en ouvrant un navigateur.

    * **Local Host:**
    Si vous utilisez Kubernetes en local, voir le guestbook en allant à l'adresse : `http://localhost:3000` dans votre navigateur

    * **Remote Host:**

    Pour voir le guestbook sur un hôte disant, trouver l'adresse IP externe et le port du load balancer dans les colonnes  **EXTERNAL-IP** et **PORTS** de la sortie de `$ kubectl get services`.

       ```console
       $ kubectl get services
       NAME           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
       guestbook      LoadBalancer   172.21.252.107   50.23.5.136   3000:31838/TCP   14m
       redis-master   ClusterIP      172.21.97.222    <none>        6379/TCP         14m
       redis-slave    ClusterIP      172.21.43.70     <none>        6379/TCP         14m
       .........
       ```

       In this scenario the URL is `http://50.23.5.136:31838`.

       Note: If no external IP is assigned, then you can get the external IP with the following command:

       ```console
       $ kubectl get nodes -o wide
       NAME           STATUS    ROLES     AGE       VERSION        EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME  
       10.47.122.98   Ready     <none>    1h        v1.10.11+IKS   173.193.92.112   Ubuntu 16.04.5 LTS   4.4.0-141-generic   docker://18.6.1
       ```

       In this scenario the URL is `http://173.193.92.112:31838`.

    2. Aller à l'adresse récupérée en sortie dans votre navigateur (par exemple : `http://50.23.5.136:31838`) . Vous devriez voir le guestbook s'afficher dans votre navigateur

       ![Guestbook](https://github.com/IBM/helm101/blob/master/docs/images/guestbook-page.png)

## Scenario 2: Déployer l'application en utilisant Helm

Dans cette partie de l'exercice, nous allons déployer l'applications en utilisant Helm. Nous définissons un nom de release à `guestbook-demo` pour le distinguer de ces précédents déploiements. Le graphique helm est disponible sur github. Cloner le dépôt[Helm 101](https://github.com/IBM/helm101) pour obtenir les fichiers :

```console
git clone https://github.com/IBM/helm101
```

Un graphique est défini comme une collection de fichiers qui décrivent un set de ressources liés de Kubernetes. Nous pouvons regarder les fichiers avant d'installer le graphique. Les fichier du graphique `guestbook` sont les suivants :

```text
.
├── Chart.yaml    \\ Un fichier YAML qui contient des informations à propos du graphique
├── LICENSE       \\ Un fichier de text qui contient la licence pour le graphique
├── README.md     \\ Un README qui fournit des informations sur l'utilisation du graphique, sa configuration, son installation, etc.
├── templates     \\ Un dossier de templates qui génère des fichiers manifest Kubernetes valide lorsqu'ils sont combinés au fichier values.yaml
│   ├── _helpers.tpl               \\ Template de définition / aide qui sont réutilisé par le graphique
│   ├── guestbook-deployment.yaml  \\ Guestbook app container ressource
│   ├── guestbook-service.yaml     \\ Guestbook app service ressource
│   ├── NOTES.txt                  \\ Un fichier texte qui contient quelques notes d'utilisation sur comment accéder à l'application après l'installation
│   ├── redis-master-deployment.yaml  \\ Redis master container ressource
│   ├── redis-master-service.yaml     \\ Redis master service ressource
│   ├── redis-slave-deployment.yaml   \\ Redis slave container ressource
│   └── redis-slave-service.yaml      \\ Redis slave service ressource
└── values.yaml   \\ Les valeurs de configuration par défaut pour le graphique
```

Note : les fichiers de templates montrés ci-dessus seront rendus dans des fichiers manifest Kubernetes avant d'être envoyé au serveur d'API de Kubernetes. Ils correspondent aux fichiers manifests que nous avons déployés avec `kubectl` (sans les fichiers helpers et de notes)

Commençons l'installation du graphique. Si l'espace de nom `helm-demo` n'existe pas, vous devez le créer avec :

```console
kubectl create namespace helm-demo
```

1. Installer l'appli avec le graphique Helm :

   ```console
   $ cd helm101/charts

   $ helm install guestbook-demo ./guestbook/ --namespace helm-demo
   NAME: guestbook-demo
   ...
   ```

   Vous devriez avoir une sortie similaire à la suivante :

   ```console
   NAME: guestbook-demo
   LAST DEPLOYED: Mon Feb 24 18:08:02 2020
   NAMESPACE: helm-demo
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   1. Get the application URL by running these commands:
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
         You can watch the status of by running 'kubectl get svc -w guestbook-demo --namespace helm-demo'
     export SERVICE_IP=$(kubectl get svc --namespace helm-demo guestbook-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
     echo http://$SERVICE_IP:3000
   ```

   L'installation du graphique lance les créations de déploiements et de services Kubernetes du maitre et des esclaves de Redis, et de l'application guestbook en une seule commande. Cela vient du fait que le graphique est une collection de fichiers qui décrivent des ressources Kubernetes liées et que Jelm gère la création de ces ressources via l'API de Kubernetes.

   Vérifier le déploiement :

   ```console
   kubectl get deployment guestbook-demo --namespace helm-demo
   ```

   Vous devriez avoir une sortie similaire à la suivante :

   ```console
   $ kubectl get deployment guestbook-demo --namespace helm-dem
   NAME             READY   UP-TO-DATE   AVAILABLE   AGE
   guestbook-demo   2/2     2            2           51m
   ```

   Pour vérifier le status des pods faisant tourner l'application, utilisez :

   ```console
   kubectl get pods --namespace helm-demo
   ```

   Vous devriez avoir une sortie similaire à la suivante :

   ```console
   $ kubectl get pods --namespace helm-demo
   NAME                            READY     STATUS    RESTARTS   AGE
   guestbook-demo-6c9cf8b9-jwbs9   1/1       Running   0          52m
   guestbook-demo-6c9cf8b9-qk4fb   1/1       Running   0          52m
   redis-master-5d8b66464f-j72jf   1/1       Running   0          52m
   redis-slave-586b4c847c-2xt99    1/1       Running   0          52m
   redis-slave-586b4c847c-q7rq5    1/1       Running   0          52m
   ```

   Pour vérifier les services, utiliser : 

   ```console
   kubectl get services --namespace helm-demo
   ```

   ```console
   $ kubectl get services --namespace helm-demo
   NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
   guestbook-demo   LoadBalancer   172.21.43.244    <pending>     3000:31367/TCP   52m
   redis-master     ClusterIP      172.21.12.43     <none>        6379/TCP         52m
   redis-slave      ClusterIP      172.21.176.148   <none>        6379/TCP         52m
   ```

1. Voir le guestbook:

   Vous pouvez maintenant jouer avec l'application guestbook en ouvrant un navigateur.

    * **Local Host:**
    Si vous utilisez Kubernetes en local, voir le guestbook en allant à l'adresse : `http://localhost:3000` dans votre navigateur

    * **Remote Host:**

    1. Pour voir le guestbook sur un hôte distant, localiser l'adresse IP externe et le port en suivant la section NOTES de la sortie de l'installation. Les commandes ressemblent aux commandes suivantes :

       ```console
       $ export SERVICE_IP=$(kubectl get svc --namespace helm-demo guestbook-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
       $ echo http://$SERVICE_IP
       http://50.23.5.136
       ```

       Combiner l'IP du service avec le port du service affiché précédemment. Dans ce scénario l'URL est :`http://50.23.5.136:31367`.

       Note: Si aucune adresse IP externe n'est assignée, vous pouvez en obtenir une avec la commande suivante ::

       ```console
       $ kubectl get nodes -o wide
       NAME           STATUS    ROLES     AGE       VERSION        EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME  
       10.47.122.98   Ready     <none>    1h        v1.10.11+IKS   173.193.92.112   Ubuntu 16.04.5 LTS   4.4.0-141-generic   docker://18.6.1
       ```

       Dans ce scénario l'URL est  `http://173.193.92.112:31367`.

    2. Aller à l'adresse récupérée en sortie dans votre navigateur (par exemple : `http://50.23.5.136:31838`) . Vous devriez voir le guestbook s'afficher dans votre navigateur

       ![Guestbook](https://github.com/IBM/helm101/blob/master/tutorial/images/guestbook-page.png?raw=true)

## Conclusion

Félicitations, vous avez déployer une application en utilisant les deux méthodes différentes de Kubernetes. Avec cet exercice, vous voyez qu'utilisez Helm demande moins de commandes à penser (en lui donnant le chemin du graphique et pas des fichiers individuels) en comparaison avec l'utilisation de `kubectl`. La gestion des application de Helm simplifie la vie de l'utilisateur.

