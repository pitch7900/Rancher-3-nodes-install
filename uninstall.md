**Désinstallation**
---------------

Désinstallation via helm
========================

````bash
helm uninstall cert-manager jetstack/cert-manager \--namespace cert-manager

helm uninstall rancher rancher-latest/rancher \--namespace cattle-system
````

Vérifications
=============

````bash
helm list --all-namespaces
````

[**<<Retour**][Home]

[Home]: /README.md
