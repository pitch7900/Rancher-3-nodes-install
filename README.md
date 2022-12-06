Principe
========

Créer un cluster Rancher en utilisant RKE2 (Rancher Kubernetes Engine) sur trois noeuds en haute disponibilité sur une installation fraiche de Ubuntu 22.04 LTS.
L'installation proprement dites de Ubuntu serveur ne sera pas détaillée ici, seul les points imports de la configuration le seront.
Important :
* Configuration en IP fixes (10.79.1.201/24 pour rancher-01, 10.79.1.202/24 pour rancher-02, 10.79.1.203/24 pour rancher-03)
* Utilisation et main mise sur un serveur de DNS interne pour changer les entrées (ici un serveur bind9 externe a cette doc)
* Connaissance de la configuration et du management du FW en front

Schéma
======

![](/media/image1.png)

Machines virtuelles
===================

* CPU : 2 (min 2)
* Mémoire : 8GB (min 4GB)
* Disque : 100GB
* OS : Ubuntu 22.04 LTS

![](/media/image2.png)

Configuration apache2 en reverse proxy
======================================

````bash
apt update
apt -y upgrade
apt -y install apache2 certbot python3-certbot-apache
a2dissite 000-default
a2enmod ssl rewrite headers proxy deflate env proxy_http negociation negotiation status proxy_balancer substitute
systemctl restart apache2
````

File :  /etc/apache2/sites-available/http.rancher.mydomain.com.conf

````apacheconf
<VirtualHost _default_:80>
  ProxyPreservehost Off
  ServerName  rancher.mydomain.com
  ServerAlias rancher.mydomain.com

  AllowEncodedSlashes NoDecode
  CustomLog ${APACHE_LOG_DIR}/access.rancher.mydomain.com_80.log combined

  <Directory /var/www/rancher>
    AllowOverride All
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/error.rancher.mydomain.com_80.log
  # Possible values include: debug, info, notice, warn, error, crit,
  # alert, emerg.
  LogLevel warn
  CustomLog ${APACHE_LOG_DIR}/access.rancher.mydomain.com_80.log combined
  DocumentRoot /var/www/rancher

RewriteEngine on
RewriteCond %{SERVER_NAME} =rancher.mydomain.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
````

````bash
mkdir -p /var/www/rancher/.well-known
echo "Options +FollowSymLinks
RewriteEngine on
RewriteCond %{REQUEST_URI} !^/.well-known/

RewriteRule (.*) https://rancher.mydomain.com/$1 [R=301,L]
">/var/www/rancher/.htaccess
chown -R www-data:www-data /var/www/
chmod 644 /var/www/rancher/.htaccess
a2ensite http.rancher.mydomain.com
systemctl reload apache2
certbot --apapche
a2dissite http.rancher.mydomain.com-le-ssl
````

File :  /etc/apache2/sites-available/ssl.rancher.mydomain.com.conf

````apacheconf
<IfModule mod_ssl.c>
<VirtualHost _default_:443>
  SSLEngine On
  SSLProxyEngine on
  SSLProxyVerify none 
  SSLProxyCheckPeerCN off
  SSLProxyCheckPeerName off


  SSLCertificateFile /etc/letsencrypt/live/rancher.mydomain.com/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/rancher.mydomain.com/privkey.pem
  Include /etc/letsencrypt/options-ssl-apache.conf
 
  RequestHeader set "X-Forwarded-Proto" expr=%{REQUEST_SCHEME}
  RequestHeader set "X-Forwarded-SSL" expr=%{HTTPS}
  RequestHeader set X-Forwarded-Port "443"
  RequestHeader set Host "rancher.mydomain.com"
  #Header edit Location "(^http[s]?://)([^/]+)" ""
  ProxyRequests Off
  ProxyPreserveHost On
  ProxyPassReverseCookieDomain kube.mydomain.local rancher.mydomain.com

  ServerName  rancher.mydomain.com
  ServerAlias rancher.mydomain.com

  ErrorLog ${APACHE_LOG_DIR}/error.rancher.mydomain.com.log
  # Possible values include: debug, info, notice, warn, error, crit,
  # alert, emerg.
  LogLevel warn
  #CustomLog ${APACHE_LOG_DIR}/access.rancher.mydomain.com.log combined

  CustomLog ${APACHE_LOG_DIR}/access.rancher.mydomain.com.log "%h %l %{SSL_CLIENT_S_DN}x  %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\""

  <Proxy "balancer://mycluster">
    BalancerMember "https://10.79.1.201:443"
    BalancerMember "https://10.79.1.202:443"
    BalancerMember "https://10.79.1.203:443"
  </Proxy>
  <Location />
    ProxyPass "balancer://mycluster/"
    ProxyPassReverse "balancer://mycluster/"
  </Location>
  
</VirtualHost>
</IfModule>
````

````bash
a2ensite ssl.rancher.mydomain.com
systemctl reload apache2
````

Configuration et préparation des serveurs Ubuntu
================================================

Installation sur ubuntu 22.04 fresh install
-------------------------------------------

--> Créer /root/.ssh/id\_rsa et /root/.ssh/autorized\_keys et les copier sur les serveurs du cluster sur le serveur rancher-01

````bash
ssh-keygen -t rsa -b 4096 -C "rancher@mydomain.local"
````

--> Au besoin [[https://linuxhint.com/generate-ssh-key-ubuntu/]](https://linuxhint.com/generate-ssh-key-ubuntu/)

Ajouter les configurations suivantes pour dire quel fichier de clef privée doit être utilisée pour atteindre chaque serveur.

````bash
echo "Host rancher-01
    Hostname rancher-01
    IdentityFile ~/.ssh/id_rsa.rancher
    IdentitiesOnly yes 
 
Host rancher-02
    Hostname rancher-02
    IdentityFile ~/.ssh/id_rsa.rancher
    IdentitiesOnly yes 

Host rancher-03
    Hostname rancher-03
    IdentityFile ~/.ssh/id_rsa.rancher
    IdentitiesOnly yes" >> /root/.ssh/config

ssh rancher-01 #Accept fingerprint and quit
ssh rancher-02 #Accept fingerprint and quit
ssh rancher-03 #Accept fingerprint and quit
 

echo "contenu du fichier /root/.ssh/id_rsa.rancher sur le serveur rancher-01">/root/.ssh/id_rsa.rancher
echo "contenu du fichier /root/.ssh/id_rsa.rancher.pub sur le serveur rancher-01" >>/root/.ssh/authorized_keys
echo "contenu du fichier  /root/.ssh/known_hosts  sur le serveur rancher-01" > /root/.ssh/known_hosts
````

Installation des packages avec parallel-ssh
-------------------------------------------

Fichier /root/rancher-hosts.txt

````txt
rancher-01
rancher-02
rancher-03
````

````bash
apt install -y pssh
parallel-ssh -t 0 -h rancher-hosts.txt apt install -y net-tools ntp bash-completion
parallel-ssh -t 0 -h rancher-hosts.txt swapoff -a
````

Activer l\'auto complétion de façon globale
---------------------------------------------

Fichier /etc/bash.bashrc et décommenter sur chaque noeud :

````bash
if ! shopt -oq posix; then
  if [ -f /usr/share/bash-completion/bash_completion ]; then
    . /usr/share/bash-completion/bash_completion
  elif [ -f /etc/bash_completion ]; then
    . /etc/bash_completion
  fi
fi
````

Configuration dns
===================

Attention on est sur une config avec du DNS round robin pour kube.mydomain.local

````txt
kube IN A 10.79.1.201
kube IN A 10.79.1.202
kube IN A 10.79.1.203
rancher-01 IN A 10.79.1.201
rancher-02 IN A 10.79.1.202
rancher-03 IN A 10.79.1.203
````

Installation et configuration cluster RKE2
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

````bash
$ kubectl get nodes

NAME STATUS ROLES AGE VERSION

rancher-01 Ready control-plane,etcd,master 88s v1.24.8+rke2r1
````

--> Tous les status doivent être \"Running\" ou \"Completed\"

````bash
root@rancher-01:/etc/rancher/rke2# kubectl get nodes
NAME         STATUS   ROLES                       AGE   VERSION
rancher-01   Ready    control-plane,etcd,master   88s   v1.24.8+rke2r1

--> Tous les status doivent être "Running" ou "Completed"
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

Ou encore :

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

### Vérifier que les nœuds s\'ajoutent bien

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

### Remettre l\'ensemble des nœuds au niveau DNS en mode round robin (config DNS)

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

Sur chaque nœud:

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

Installation de Helm
====================

Installation
------------

[[https://helm.sh/docs/intro/install/]](https://helm.sh/docs/intro/install/)

````bash
parallel-ssh -t 0 -h rancher-hosts.txt curl -fsSL -o /root/get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
parallel-ssh -t 0 -h rancher-hosts.txt chmod +x /root/get_helm.sh
parallel-ssh -t 0 -h rancher-hosts.txt /root/get_helm.sh
````

Vérification
------------

````bash
helm version --client
helm ls --all-namespaces
````

Ajout de l\'auto-complétion Helm
--------------------------------

````bash
parallel-ssh -t 0 -h rancher-hosts.txt "helm completion bash > /etc/bash_completion.d/helm"
````

*Lien : \<[[https://helm.sh/docs/helm/helm\_completion\_bash/]](https://helm.sh/docs/helm/helm_completion_bash/)\>*

Installation de cert-manager
============================

[[https://cert-manager.io/docs/installation/helm/\#3-install-customresourcedefinitions]](https://cert-manager.io/docs/installation/helm/#3-install-customresourcedefinitions)

````bash
parallel-ssh -t 0 -h rancher-hosts.txt helm repo add jetstack https://charts.jetstack.io
parallel-ssh -t 0 -h rancher-hosts.txt helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true


kubectl -n cert-manager rollout status deploy/cert-manager
````

Installation de Rancher
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

Troubleshooting
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

Désinstallation via helm
========================

 

Désinstallation
---------------


````bash
helm uninstall cert-manager jetstack/cert-manager \--namespace cert-manager

helm uninstall rancher rancher-latest/rancher \--namespace cattle-system
````

Vérifications
=============

````bash
helm list --all-namespaces
````

Ressources
==========

[[https://docs.ranchermanager.rancher.io/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher]](https://docs.ranchermanager.rancher.io/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher)

Installation de rke2 (doc mieux faite que l\'officielle)

[[https://infohub.delltechnologies.com/l/suse-rancher-and-rke2-kubernetes-cluster-in-apex-private-cloud-services/steps-to-install-rke2-cluster-three-nodes-manually]](https://infohub.delltechnologies.com/l/suse-rancher-and-rke2-kubernetes-cluster-in-apex-private-cloud-services/steps-to-install-rke2-cluster-three-nodes-manually)

Configuration file (ne sert à RIEN sur le site) pour installation de RKE2 --> Voir le site de Dell

[[https://docs.rke2.io/install/configuration\#configuration-file]](https://docs.rke2.io/install/configuration#configuration-file)

Un peu plus d\'informations sur :

[[https://docs.rke2.io/reference/server\_config]](https://docs.rke2.io/reference/server_config)

[[https://docs.ranchermanager.rancher.io/pages-for-subheaders/installation-and-upgrade]](https://docs.ranchermanager.rancher.io/pages-for-subheaders/installation-and-upgrade)

Rancher Rodeo :

[[https://9to5tutorial.com/rodeo-scenario-with-v2-6-2021-10]](https://9to5tutorial.com/rodeo-scenario-with-v2-6-2021-10)

Helm

[[https://helm.sh/docs/intro/install/]](https://helm.sh/docs/intro/install/)

Rancher-cli

[[https://docs.ranchermanager.rancher.io/v2.6/reference-guides/cli-with-rancher/rancher-cli\#commands]](https://docs.ranchermanager.rancher.io/v2.6/reference-guides/cli-with-rancher/rancher-cli#commands)
