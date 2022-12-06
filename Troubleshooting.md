**Troubleshooting**
===============

Les configurations se trouvent ici : /var/lib/rancher/rke2/server/manifests/

*Lien : \<[[https://github.com/rancher/rke2/issues/1446]](https://github.com/rancher/rke2/issues/1446)\>*

[[https://docs.ranchermanager.rancher.io/v2.6/reference-guides/cli-with-rancher/rancher-cli\#commands]](https://docs.ranchermanager.rancher.io/v2.6/reference-guides/cli-with-rancher/rancher-cli#commands)

````bash
kubectl get pods -n cattle-system -o wide
NAME                               READY   STATUS    RESTARTS   AGE
rancher-5845f6ffc7-vmpd8           1/1     Running   0          100m
rancher-webhook-5f46f4f7df-l2jzh   1/1     Running   0          84m

kubectl -n cattle-system logs rancher-5845f6ffc7-vmpd8


kubectl get pods --all-namespaces

kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get pods --all-namespaces
````

Pour voir les connexions ingress (entrantes)

````bash
kubectl get ingress --all-namespaces
NAMESPACE       NAME      CLASS    HOSTS                      ADDRESS                               PORTS   AGE
cattle-system   rancher   <none>   rancher.mydomain.com   10.79.1.201,10.79.1.202,10.79.1.203   80      10h
 ````

[**<<Retour**][Home]

[Home]: /README.md
