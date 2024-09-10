Ce depôt accompagne mes formations autour de Kubernetes

Exercice supplémentaires :

- StatefulSet : https://kubernetes.io/docs/tutorials/stateful-application/cassandra/#using-a-statefulset-to-create-a-cassandra-ring 
- Ingress sur Minikube : https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/
- RBAC : https://cours.hadrienpelissier.fr/03-kubernetes/tp_opt_k8s_rbac/ 


# Initialiser un cluster sur labs

URL : https://labs.play-with-k8s.com/ 

* Se connecter
* Créer une première instance
* Passer les commandes :
```
kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
kubeadm token create --print-join-command
```
* Créer une seconde instance et passer la commande donnée par la dernière commande précédente, celle ci ressemble à  :
```
 kubeadm join 192.168.0.8:6443 --token cuorwh.1j6budao6syl45go --discovery-token-ca-cert-hash sha256:tokencrypte
```

* Revenir sur la premiere instance puis passer la commande suivante :
```
kubectl label node node2 node-role.kubernetes.io/worker=worker1
```
* Vous devriez avoir deux noeuds opérationnels
