**Installation de Helm**
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

[**<<Retour**][Home]

[Home]: /README.md
