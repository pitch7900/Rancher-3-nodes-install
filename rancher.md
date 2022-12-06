**Installation de Rancher**
=======================

Installation de Rancher avec Helm et un load-balancer L7 externe (TLS terminaison sur le LB)

Ajout du repository via helm
----------------------------

````bash
parallel-ssh -t 0 -h rancher-hosts.txt helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
parallel-ssh -t 0 -h rancher-hosts.txt helm repo update

````

*Lien : \<[[https://docs.ranchermanager.rancher.io/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster\#1-add-the-helm-chart-repository]](https://docs.ranchermanager.rancher.io/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster#1-add-the-helm-chart-repository)\>*

Création du namespace pour rancher
----------------------------------

````bash
kubectl create namespace cattle-system
````

Install Rancher (2 réplicas)
----------------------------

````bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.mydomain.com\
  --set replicas=2 \
  --set tls=external \
  --create-namespace


 

NAME: rancher
LAST DEPLOYED: Wed Nov 30 12:14:20 2022
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.
NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.
Check out our docs at https://rancher.com/docs/
If you provided your own bootstrap password during installation, browse to https://rancher.mydomain.com to get started.
If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:
```
echo https://rancher.mydomain.com/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```
To get just the bootstrap password on its own, run:
```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

Happy Containering!

````

Vérification
------------

````bash
while true; do curl -kv https://rancher.mydomain.com 2>&1 | grep -q "workloads"; if [ $? != 0 ]; then echo "Rancher isn't ready yet"; sleep 5; continue; fi; break; done; echo "Rancher is Ready";

kubectl -n cattle-system get pods
kubectl -n cattle-system describe pod
````

*Lien : \<[[https://docs.ranchermanager.rancher.io/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/troubleshooting]](https://docs.ranchermanager.rancher.io/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/troubleshooting)\>*

Ça peut être long...

Install Rancher CLI
-------------------

[[https://docs.ranchermanager.rancher.io/v2.6/reference-guides/cli-with-rancher/rancher-cli\#commands]](https://docs.ranchermanager.rancher.io/v2.6/reference-guides/cli-with-rancher/rancher-cli#commands)

````bash
parallel-ssh -t 0 -h rancher-hosts.txt wget https://github.com/rancher/cli/releases/download/v2.7.0/rancher-linux-amd64-v2.7.0.tar.gz -O /root/rancher_cli.tgz

parallel-ssh -t 0 -h rancher-hosts.txt tar -zxvf /root/rancher_cli.tgz
parallel-ssh -t 0 -h rancher-hosts.txt cp /root/rancher-v2.7.0/rancher /usr/bin/rancher
parallel-ssh -t 0 -h rancher-hosts.txt chmod +x /usr/bin/rancher
````

[**<<Retour**][Home]

[Home]: /README.md
