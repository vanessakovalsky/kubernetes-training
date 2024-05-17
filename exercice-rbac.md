# Gestion des rôles et des droits


## Création d'un ServiceAccount

* Nous allons commencer à manipuler les fonctionnalités de sécurité de Kubernetes en réalisant une opération courante : la création d'un ServiceAccount restreint à un namespace, ainsi que la création d'un kubeconfig associé.

    * Commencer par créer le ServiceAccount
```
kubectl create serviceaccount <nom du compte>
```
* Pour créer le kubeconfig, nous allons avoir besoin de consulter le Secret généré pour le compte.

```
kubectl get sa <nom du compte> -o yaml
kubectl get secret <nom du secret> -o yaml
```
  * Récupérer les valeurs token et ca.crt dans le secret, décoder le base64 et enregister ca.crt dans un fichier
  * Récupérer l'url du cluster dans la configuration actuelle (kubectl config view)

*Nous avons tous les éléments en main pour générer un nouveau kubeconfig, en indiquant un chemin vers un fichier inexistant avec l'option --kubeconfig.

  * Créer l'entrée cluster du kubeconfig
```
kubectl config set-cluster <nom du cluster> \
  --kubeconfig=<chemin vers le nouveau kubeconfig>
  --server=<adresse du server>
  --certificate-authority=<chemin vers ca.crt>
  --embed-certs=true
```
  * Créer l'entrée utilisateur du kubeconfig

```
kubectl config set-credentials <nom de utilisateur> \
  --kubeconfig=<chemin vers le nouveau kubeconfig>
  --token=<token du ServiceAccount>
```
  * Enfin, créer un contexte
```
kubectl config set-context <nom du contexte> \
  --kubeconfig=<chemin vers le nouveau kubeconfig>
  --cluster=<nom du cluster>
  --user=<nom de l'utilisateur>
  --namespace=<nom du namespace>
```
* Afficher le contenu du nouveau fichier kubeconfig, celui-ci doit être complet.

## Ajout de droits avec les Roles

* Nous allons donner quelques droits au ServiceAccount nouvellement créé
  * Compléter le manifeste suivant pour créer un Role qui autorise toutes les opérations sur toutes les ressources sauf les PersistentVolumeClaims qui ne doivent pas pouvoir être supprimés.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-all-pvc-ro
rules:
  - apiGroups:
      - ""
      - apps
      - autoscaling
      - batch
      - extensions
      - policy
      - rbac.authorization.k8s.io
    resources:
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs:
      - get
      - watch
      - list
```
  * Créer un RoleBinding pour appliquer le ClusterRole au niveau du namespace.
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <nom du RoleBinding>
  namespace: <nom du namespace>
subjects:
  - kind: ServiceAccount
    name: <nom du ServiceAccount
roleRef:
  apiGroup: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  name: <nom du ClusterRole
```
  * Pour tester, le plus simple est d'utiliser l'option --as de kubectl pour exécuter une commande en tant qu'un autre utilisateur.

* Nous venons de lier un ClusterRole avec un RoleBinding (et non pas un ClusterRoleBinding).

* Que se passe-t-il dans ce cas particulier ?

## Agrégation de ClusterRoles

* L'API RBAC offre la possibilité d'agréger plusieurs ClusterRoles dans un ClusterRole "parapluie", ce qui peut simplifier la gestion des droits.

* Nous allons créer les ClusterRole suivant
  * Un ClusterRole pod-viewer capable d'afficher les Pods
  * Un ClusterRole service-viewer capable d'afficher les Services
  * Un ClusterRole parapluie qui va agréger les deux précédents

* L'agrégation fonctionne par sélection de label.

   * Créer le ClusterRole parapluie.
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: reviewer
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      aggregate-to-reviewer: "true"
rules: []
```
   * Créer les ClusterRole à agréger, selon le modèle suivant.
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ...
  labels:
    ...
rules:
  ...
```

## NetworkSecurityPolicies

* Pour cet exercice, nous allons redéployer l'application echo-server ainsi que son Service, ceci dans 2 namespaces différents. L'objectif sera de bloquer tout le trafic entrant dans un premier temps, puis ouvrir l'accès au seul port du service echo-server.

   * Commencer par déployer l'application echo-server dans les deux namespace
   * Se connecter à l'intérieur du Pod du premier namespace et requêter l'echo-server de l'autre namespace.
```
kubectl exec -it <nom du pod> -n <premier namespace> /bin/sh
curl echo-server.<second namespace>.svc.cluster.local
```
* La requête doit s'exécuter avec succès.

* Appliquer la NetworkSecurityPolicy par défaut dans le second namespace.
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```
* À partir de maintenant, tout le trafic entrant doit être bloqué dans le second namespace (reproduire le test pour vérifier).

* Il ne reste qu'à créer une autre NetworkSecurityPolicy pour ouvrir l'accès au port 8080 de l'echo-server du second namespace.
   * Appliquer le manifeste suivant

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-echo-server
spec:
  podSelector:
    matchLabels:
      ...
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          ...
    ports:
    - protocol: TCP
      port: ...
```
   * Ajouter tous les labels nécessaires

* Reproduire le test précédent, la requête doit fonctionner.

* Pour vérifier que le reste du trafic est bien bloqué, on peut changer le port de l'echo-server et reproduire le test.


