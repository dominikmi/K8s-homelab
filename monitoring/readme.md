!!!Prometheus & Grafana setup

1) Create manually Storage Class definition: `manual-storage-class.yaml`
2) Create two PV: storage-volume & config-volume,
3) Install Prometheus `helm install prometheus stable/prometheus --set server.storageClass=manual`
4) In `monitoring` forlder mkdir `grafana`
6) cd `grafana` and create two files:
	- grafana-configmap.yaml
	- grafana-values.yaml
7) apply the configmap for Grafana: `kubectl apply -f grafana-configmap.yaml`
8) Install Grafana `helm install grafana stable/grafana -f grafana-values.yaml --namespace monitoring`
