**Installation de cert-manager**
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

[**<<Retour**][Home]

[Home]: /README.md
