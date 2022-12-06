Principe
========

Créer un cluster Rancher en utilisant RKE2 (Rancher Kubernetes Engine) sur trois noeuds en haute disponibilité sur une installation fraiche de Ubuntu 22.04 LTS.
L'installation proprement dites de Ubuntu serveur ne sera pas détaillée ici, seul les points imports de la configuration le seront.
Important :

* Configuration en IP fixes :

  * 10.79.1.201/24 pour rancher-01
  * 10.79.1.202/24 pour rancher-02
  * 10.79.1.203/24 pour rancher-03

* Utilisation et main mise sur un serveur de DNS interne pour changer les entrées (ici un serveur bind9 externe à cette doc)
* Connaissance de la configuration et du management du FW en front

Index
=====

* [Schéma](/Scheme.md)
* [Machines virtuelles](/virtualmachines.md)
* [Configuration apache2 en reverse proxy](/apache2reverseproxy.md)
* [Configuration et préparation des serveurs Ubuntu](/ubuntuservers.md)
* [Installation et configuration cluster RKE2](/rke2.md)
* [Installation de Helm](/helm.md)
* [Installation de cert-manager](/cert-manager.md)
* [Installation de Rancher](/rancher.md)
* [Troubleshooting](/Troubleshooting.md)
* [Désinstallation](/uninstall.md)
* [Ressources et liens externes](/ressources.md)
