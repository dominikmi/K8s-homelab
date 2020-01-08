## Longhorn deployment on local cluster

1) Download longhorn helm chart package with `git clone https://github.com/longhorn/longhorn.git`
2) Change kind to NodePort in the Service definition in the *values.yaml*
3) Create *longhorn-system* namespace: `kubectl create longhorn-system`
4) Get to the longhorn helm chart folder, install longhorn with Helm: `helm install longhorn . --namespace longhorn-system`
