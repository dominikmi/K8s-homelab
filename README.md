# K8s-stuff
I needed a local, totally cloud independent (that sounds abhorrent!) environment that I can try, break, destroy, fix, start all over again, leave it for a night or a week, without paying attention how much *$$$* i already owe a cloud provider. This is a repository of all my test deployments I have done in my home Kubes lab. It contains various homebrewed or modified vendor yaml files.
Nothing fancy so far.  Basic stuff.

- [setting up a Kubernetes lab on Fedora31](kube-installation/README.md)
- [adding-roles-to-users (exercise to define certain RBAC for given users)](adding-roles-to-users/README.md)
- [app-deployments (test apps deployments accessible via ingress nginx)](app-deployments/README.md)
- [longhorn (NodePort longhorn deployment)](longhorn/README.md)
- [metrics-server (basic metrics for K8s HPA)](metrics-server/README.md)
- [monitoring (Prometheus & Grafana deployment)](monitoring/README.md)
- [nginx-ingress (my workout on deploying local ingress)](nginx-ingress/README.md)
- [wordpress (some database, PVC-PV exercise, longhorn tests)](wordpress/README.md)

# ToDo

- tutorial on setting up helm v3 along with a Github repo,
- working with Kube DNS,
- setting up all web services access through nginx ingress,
- setting up nginx ingress controller by a helm chart,
- application deployment using helm charts,
- test istio deployments,
- setup audit log policy & transport logs via fluentd to some LM tool,
- deploy automated CI/CD.
