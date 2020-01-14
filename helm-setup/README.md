## Helm setup

1. To get Helm 3 installed, simply follow the [tutorial](https://helm.sh/docs/intro/install/) how to install and how to [use](https://helm.sh/docs/intro/using_helm/) helm.
2. Once you get helm set in your host, it will start using your credentials `~/.kube/config` to communicate with Kubes APIs.
3. Somewhere in your work folder create a `<local-helm-charts-folder>`, go there and start searching charts for stuff you like with `helm search hub <name>.
4. You basically add a repo like you'd do with a repo for your linux packages: `helm repo add <local-repo-name> <repo URL>` (optionally you can also pass there login and password to the repo, if it's private one)
	```
	$ helm repo list
	NAME      	URL                                                               
	stable    	https://kubernetes-charts.storage.googleapis.com                  
	myhelmrepo	https://raw.githubusercontent.com/dominikmi/my-helm-charts/master/

5. I pulled down the following charts from the stable repository:
	- nginx-ingress: `helm fetch stable/nginx-ingress`
	- longhorn (this is an exception): `git clone https://github.com/longhorn/longhorn.git`
	- prometheus: `helm fetch stable/prometheus`
	- weavescope: `helm fetch stable/weave-scope`
	- grafana: `helm fetch stable/grafana`

6. After pulling down stuff you like, you end up with bunch of **.tgz** files in one folder.
7. Unpack them all and you'll see that each chart folder has pretty much the same structure. More you can read in this excellent [tutorial](https://docs.bitnami.com/kubernetes/how-to/create-your-first-helm-chart/).
8. For the purpose of getting my tasks done, all I needed to configure was just playing around with the `values.yaml` files. So I made a copy of it and .. played. Then you deploy the thing, to see how it works along with the values you put in your my-values.yaml. Use the following command: `helm install <your-name-of-the-deployment> -f my-values.yaml .` Optionally, you can specify a Kubernetes namespace you want this thing to be deployed at.
9. Check the deployment with `helm list -n <namespace>` example:
	```
	NAME      	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART            	APP VERSION
	grafana   	monitoring	3       	2020-01-14 23:07:43.530761188 +0100 CET	deployed	grafana-4.3.0    	6.5.2      
	prometheus	monitoring	1       	2020-01-11 19:01:26.205795485 +0100 CET	deployed	prometheus-9.7.4 	2.13.1     
	weavescope	monitoring	3       	2020-01-14 22:03:10.222933597 +0100 CET	deployed	weave-scope-1.1.8	1.12.0     
        
10. If you feel like you need to modify something else as well or fix incorrect value do: `helm upgrade <name> -f my-values.yaml . -n <namespace>`, check again and you'll see that the revision number in the REVISION column has changed. Of course, you can check the history of a given deployment and eventually roll back if needed. Example: 

	```
	$ helm history grafana -n monitoring
	REVISION	UPDATED                 	STATUS    	CHART        	APP VERSION	DESCRIPTION     
	1       	Thu Jan  9 22:16:41 2020	superseded	grafana-4.3.0	6.5.2      	Install complete
	2       	Tue Jan 14 22:44:12 2020	superseded	grafana-4.3.0	6.5.2      	Upgrade complete
	3       	Tue Jan 14 23:07:43 2020	deployed  	grafana-4.3.0	6.5.2      	Upgrade complete

11. If you need to destroy and wipe out completely the deployment you did with helm do: `helm delete <name> -n namespace`
--------------------
## Helm chart private repository setup

[WIP]
