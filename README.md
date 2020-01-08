# K8s-stuff
This is a repository of all my test deployments on my home K8s. It contains various homebrewed or modified vendor yaml files.
Nothing fancy, so far. Basic stuff.

- adding-roles-to-users (exercise to define certain RBAC for given users)
- app-deployments (test apps deployments accessible via ingress nginx)
- longhorn (NodePort longhorn deployment)
- metrics-server (basic metrics for K8s HPA)
- monitoring (Prometheus & Grafana deployment)
- nginx-ingress (my workout on deploying local ingress)
- wordpress (some database, PVC-PV exercise, longhorn tests)

# ToDo

- converting all generic yaml deployment to HELM charts stored at the github monorepo,
- a tutorial on setting up fully fledged home K8s installation (in a model of 1+2) on 16GB 6cores laptop with qemu/kvm,
- test istio deployments,
- setup audit log policy & transport logs via fluentd to some LM tool,
- deploy automated CI/CD.
