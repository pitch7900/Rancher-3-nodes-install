**Installation et configuration cluster RKE2**
==========================================

Désactiver le round robin DNS sur les noeuds 02 et 03.

````txt
kube IN A 10.79.1.201
;kube IN A 10.79.1.202
;kube IN A 10.79.1.203
````

````bash
parallel-ssh -t 0 -h rancher-hosts.txt mkdir -p /etc/rancher/rke2/
parallel-ssh -t 0 -h rancher-hosts.txt "curl -sfL https://get.rke2.io | sh -"
````

Sur le premier serveur (rancher-01)
--------------------------------------

````bash
echo "token: <SharedToken>
write-kubeconfig-mode: "0644"
tls-san:
  - kube.mydomain.local
  - rancher-01.mydomain.local
  - rancher-02.mydomain.local
  - rancher-03.mydomain.local
" > /etc/rancher/rke2/config.yaml

root@rancher-01:/etc/rancher/rke2# systemctl enable --now rke2-server.service
````

Attendre que le SERVICE monte sur le premier serveur...

Vérification des noeuds (STATUS "Ready" nécessaire)

````bash
root@rancher-01:/etc/rancher/rke2# kubectl get nodes
NAME         STATUS   ROLES                       AGE   VERSION
rancher-01   Ready    control-plane,etcd,master   88s   v1.24.8+rke2r1
````

Tous les status des pods doivent être "Running" ou "Completed"

````bash
root@rancher-01:/etc/rancher/rke2# kubectl get pods -A
NAMESPACE     NAME                                                    READY   STATUS      RESTARTS   AGE
kube-system   cloud-controller-manager-rancher-01                     1/1     Running     18         2m20s
kube-system   etcd-rancher-01                                         1/1     Running     5          109s
kube-system   helm-install-rke2-canal-snr6k                           0/1     Completed   0          2m17s
kube-system   helm-install-rke2-coredns-clznq                         0/1     Completed   0          2m17s
kube-system   helm-install-rke2-ingress-nginx-2shqq                   0/1     Completed   0          2m17s
kube-system   helm-install-rke2-metrics-server-7pxsp                  0/1     Completed   0          2m17s
kube-system   kube-apiserver-rancher-01                               1/1     Running     10         2m19s
kube-system   kube-controller-manager-rancher-01                      1/1     Running     16         2m22s
kube-system   kube-proxy-rancher-01                                   1/1     Running     5          2m18s
kube-system   kube-scheduler-rancher-01                               1/1     Running     11         2m22s
kube-system   rke2-canal-7lh8r                                        2/2     Running     0          2m4s
kube-system   rke2-coredns-rke2-coredns-58fd75f64b-kqvr8              1/1     Running     0          2m8s
kube-system   rke2-coredns-rke2-coredns-autoscaler-768bfc5985-m9sbp   1/1     Running     0          2m8s
kube-system   rke2-ingress-nginx-controller-zl4cr                     1/1     Running     0          50s
kube-system   rke2-metrics-server-67697454f8-ktgbm                    1/1     Running     0          67s

````

Ou encore il faut qu'il y ait 11 pods Running :

````bash
kubectl get pods -A | grep Running | wc -l
11
````

Sur les deux autre serveurs (rancher-02 et rancher-03)
--------------------------------------------------------

````bash
echo "server: https://kube.mydomain.local:9345
write-kubeconfig-mode: "0644"
token: <SharedToken>
tls-san:
  - kube.mydomain.local
  - rancher-01.mydomain.local
  - rancher-02.mydomain.local
  - rancher-03.mydomain.local
"> /etc/rancher/rke2/config.yaml
````

### Enregistrer et démarrer le service

````bash
systemctl enable --now rke2-server.service
````

[[https://docs.rke2.io/upgrade/manual\_upgrade]](https://docs.rke2.io/upgrade/manual_upgrade)

### Vérifier que les noeuds s'ajoutent bien

````bash
$ kubectl get nodes
NAME         STATUS   ROLES                       AGE   VERSION
rancher-01   Ready    control-plane,etcd,master   38m   v1.24.8+rke2r1
rancher-02   Ready    control-plane,etcd,master   25m   v1.24.8+rke2r1
rancher-03   Ready    control-plane,etcd,master   21m   v1.24.8+rke2r1
````

On peut aussi vérifier l\'état des pods déployés:

````bash
kubectl get pods -A -o wide
NAMESPACE     NAME                                                    READY   STATUS      RESTARTS         AGE     IP            NODE         NOMINATED NODE   READINESS GATES
kube-system   cloud-controller-manager-rancher-01                     1/1     Running     18               38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   cloud-controller-manager-rancher-02                     1/1     Running     1 (2m17s ago)    2m14s   10.79.1.202   rancher-02   <none>           <none>
kube-system   cloud-controller-manager-rancher-03                     1/1     Running     16               20m     10.79.1.203   rancher-03   <none>           <none>
kube-system   etcd-rancher-01                                         1/1     Running     5                38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   etcd-rancher-02                                         1/1     Running     7 (3m9s ago)     25m     10.79.1.202   rancher-02   <none>           <none>
kube-system   etcd-rancher-03                                         1/1     Running     6                20m     10.79.1.203   rancher-03   <none>           <none>
kube-system   helm-install-rke2-canal-snr6k                           0/1     Completed   0                38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   helm-install-rke2-coredns-clznq                         0/1     Completed   0                38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   helm-install-rke2-ingress-nginx-2shqq                   0/1     Completed   0                38m     10.42.0.4     rancher-01   <none>           <none>
kube-system   helm-install-rke2-metrics-server-7pxsp                  0/1     Completed   0                38m     10.42.0.3     rancher-01   <none>           <none>
kube-system   kube-apiserver-rancher-01                               1/1     Running     10               38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   kube-apiserver-rancher-02                               1/1     Running     11 (3m10s ago)   25m     10.79.1.202   rancher-02   <none>           <none>
kube-system   kube-apiserver-rancher-03                               1/1     Running     7                20m     10.79.1.203   rancher-03   <none>           <none>
kube-system   kube-controller-manager-rancher-01                      1/1     Running     16               38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   kube-controller-manager-rancher-02                      1/1     Running     12 (2m21s ago)   25m     10.79.1.202   rancher-02   <none>           <none>
kube-system   kube-controller-manager-rancher-03                      1/1     Running     16               20m     10.79.1.203   rancher-03   <none>           <none>
kube-system   kube-proxy-rancher-01                                   1/1     Running     5                38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   kube-proxy-rancher-02                                   1/1     Running     0                2m1s    10.79.1.202   rancher-02   <none>           <none>
kube-system   kube-proxy-rancher-03                                   1/1     Running     6                20m     10.79.1.203   rancher-03   <none>           <none>
kube-system   kube-scheduler-rancher-01                               1/1     Running     11               38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   kube-scheduler-rancher-02                               1/1     Running     7 (3m10s ago)    25m     10.79.1.202   rancher-02   <none>           <none>
kube-system   kube-scheduler-rancher-03                               1/1     Running     11               20m     10.79.1.203   rancher-03   <none>           <none>
kube-system   rke2-canal-4rpfj                                        2/2     Running     0                2m10s   10.79.1.202   rancher-02   <none>           <none>
kube-system   rke2-canal-7lh8r                                        2/2     Running     0                38m     10.79.1.201   rancher-01   <none>           <none>
kube-system   rke2-canal-szcms                                        2/2     Running     0                21m     10.79.1.203   rancher-03   <none>           <none>
kube-system   rke2-coredns-rke2-coredns-58fd75f64b-kqvr8              1/1     Running     0                38m     10.42.0.5     rancher-01   <none>           <none>
kube-system   rke2-coredns-rke2-coredns-58fd75f64b-wxkj8              1/1     Running     0                25m     10.42.2.2     rancher-03   <none>           <none>
kube-system   rke2-coredns-rke2-coredns-autoscaler-768bfc5985-m9sbp   1/1     Running     0                38m     10.42.0.2     rancher-01   <none>           <none>
kube-system   rke2-ingress-nginx-controller-74nzv                     1/1     Running     0                20m     10.42.2.3     rancher-03   <none>           <none>
kube-system   rke2-ingress-nginx-controller-qdxr9                     1/1     Running     0                80s     10.42.1.2     rancher-02   <none>           <none>
kube-system   rke2-ingress-nginx-controller-zl4cr                     1/1     Running     0                37m     10.42.0.8     rancher-01   <none>           <none>
kube-system   rke2-metrics-server-67697454f8-ktgbm                    1/1     Running     0                37m     10.42.0.6     rancher-01   <none>           <none>

````

### Remettre l\'ensemble des noeuds au niveau DNS en mode round robin (config DNS)

````txt
kube IN A 10.79.1.201
kube IN A 10.79.1.202
kube IN A 10.79.1.203
````

Export de kubctl pour utilisation globale
-----------------------------------------

````bash
parallel-ssh -t 0 -h rancher-hosts.txt cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin
parallel-ssh -t 0 -h rancher-hosts.txt ln -s /usr/local/bin/kubectl /usr/local/bin/k
parallel-ssh -t 0 -h rancher-hosts.txt mkdir -p ~/.kube
parallel-ssh -t 0 -h rancher-hosts.txt ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
````

Auto-complétion pour Kubectl
----------------------------

````bash
parallel-ssh -t 0 -h rancher-hosts.txt "kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null"
````

*Lien : \<[[https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/]](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/)\>*

Check des services
--------------------

````bash
kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
NAME         STATUS   ROLES                       AGE   VERSION
rancher-01   Ready    control-plane,etcd,master   76m   v1.20.15+rke2r2
rancher-02   Ready    control-plane,etcd,master   23m   v1.20.15+rke2r2
rancher-03   Ready    control-plane,etcd,master   24m   v1.20.15+rke2r2
````

Sur chaque noeud:

````bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml" >> /root/.bashrc
````

Vérification de l\'état du déploiement
--------------------------------------

````bash
kubectl get componentstatuses
kubectl get nodes


$ kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
$ kubectl get nodes
NAME         STATUS   ROLES                       AGE   VERSION
rancher-01   Ready    control-plane,etcd,master   77m   v1.20.15+rke2r2
rancher-02   Ready    control-plane,etcd,master   24m   v1.20.15+rke2r2
rancher-03   Ready    control-plane,etcd,master   25m   v1.20.15+rke2r2
````

[**<<Retour**][Home]

[Home]: /README.md
