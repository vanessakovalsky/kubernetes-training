# Exercice 4. Partage des graphiques Helm

Un des aspects clés de la fourniture d'application signifie la partager avec d'autres. Le partage peut être une utilisation directe (par des utilisateurs ou des pipelines CI/CD) ou en tant que dépendances pour d'autre graphiques. Si les gens ne peuvent pas trouver l'application, ils ne peuvent pas l'utiliser.

Une façon de parager ces graphiques est un dépôt, qui est une localisation ou les graphiques packagés peuvent être stockés et partagés. 

## Utiliser des graphiques depuis un dépôts public 

Les graphiques Helm peuvent être disponible sur un depôt distant ou dans un environnement / dépôt local. Les dépôts distants peuvent être public comme [Bitnami Charts](https://github.com/bitnami/charts) ou [IBM Helm Charts](https://github.com/IBM/charts), ou des dépôts hergés comme ceux de  Google Cloud Storage ou GitHub. Voir le [Helm Chart Repository Guide](https://helm.sh/docs/topics/chart_repository/) pour plus d'informations. Vous pouvez en apprendre plus sur la structure d'un dépôt de graphique en étudiant le [chart index file](../../index.yaml) de ce dépôt.

Dans cet exercice, nous verrons comment installer le graphique `guestbook` depuis [Helm101 repo](https://ibm.github.io/helm101/).

1. Vérifier les depôts configurés sur votre système :

   ```console
   helm repo list
   ```

   Vous devriez avoir un retour similaire au suivant :

   ```console
   $ helm repo list
   Error: no repositories to show
   ```

   > Note: Les depôts de graphique ne sont pas installés par défaut avec Helm v3. Il est nécessaire que vous ajoutiez les deoôts pour les graphiques que vous voulez utilisez. Le [Helm Hub](https://hub.helm.sh) fournit une recherche centralisé pour les graphiques publics distribués. Utilisant le hub pour identifier le graphique et le dépôts qui l'heberge et l'ajouter à votre liste de dépôt locale. 
   
1. Ajouter le dépôt `helm101`:

   ```console
   helm repo add helm101 https://ibm.github.io/helm101/
   ```

   Vous devriez avoir un retour similaire au suivant :

   ```console
   $ helm repo add helm101 https://ibm.github.io/helm101/
   "helm101" has been added to your repositories
   ```

   Vous pouvez aussi chercher dans vos dépôts les graphiques en utilisant la commande suivante :

   ```console
   helm search repo helm101
   ```

   ```console
   $ helm search repo helm101
   NAME              CHART VERSION  APP VERSION DESCRIPTION
   helm101/guestbook 0.2.1                      A Helm chart to deploy Guestbook three tier web...
   ```

1. Installer le graphique

   Comme expliqué, nous allons installer le graphique `guestbook` depuis [Helm101 repo](https://ibm.github.io/helm101/). Comme le dépôt est ajouté à votre liste de dépôts locaux, on peut récupérer le graphique en utilisant `repo name/chart name`, en d'autres termes `helm101/guestbook`. Pour le voir en action, nous installerons l'application dans un nouvel espace de nom appelé `repo-demo`.

   Si l'espace de nom `repo-demo` n'existe pas, créer le en utilisant :

   ```console
   kubectl create namespace repo-demo
   ```

   Installons le graphique en utilisant la commande :

   ```console
   helm install guestbook-demo helm101/guestbook --namespace repo-demo
   ```

   Vous devriez avoir un retour similaire au suivant :

   ```console
   $helm install guestbook-demo helm101/guestbook --namespace repo-demo
   NAME: guestbook-demo
   LAST DEPLOYED: Tue Feb 25 15:40:17 2020
   NAMESPACE: repo-demo
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   1. Get the application URL by running these commands:
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get svc -w guestbook-demo --namespace repo-demo'
     export SERVICE_IP=$(kubectl get svc --namespace repo-demo guestbook-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
     echo http://$SERVICE_IP:3000
   ```

   Vérifiez que la release s'est déployée comme attendu avec la commande suivante :

   ```console
   $ helm list -n repo-demo
   NAME           NAMESPACE   REVISION UPDATED                                   STATUS   CHART            APP VERSION
   guestbook-demo repo-demo   1        2020-02-25 15:40:17.627745329 +0000 UTC   deployed guestbook-0.2.1
   ```

## Conclusion

Cette exercice permet une première approche des dépôts Helm pour voir comment les graphiques peuvent être installés. La possibilité de partager vos graphiques facilite l'utilisation pour vous et vos clients.
