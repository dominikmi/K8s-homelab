
# Setting up nginx as ingress on K8s on-prem deployment.

## Cluster: master + 2 workers:

- kmaster:  192.168.122.100
- kworker1: 192.168.122.101
- kworker2: 192.168.122.102

## Task

We would like to test an app deployment, where the app will has 2 replicas accessible via port 80 defined in the app service. The nginx deployment will be our load balancer object with default back-end. All ingress traffic coming to our apps as app.dominik.com/app1 or ../app2 will be routed to the respective deployed app.

- Applications namespace: aplikacja
- Nginx and ingress namespace: ingress

The app deployment is described [here](https:///github.com/dominikmi/K8s-stuff/tree/master/app-deployment/README.md)

1. Create ingress namespace: `kubectl create namespace ingress`
2. Create default backend for nginx - service (to expose the nginx to the net) and deployment definitions:

- file: **default-backend-deploy.yaml**
- file: **default-backend-svc.yaml**

3. Let's deploy both (mark the namespace defined in both):
- `kubectl apply -f default-backend-svc.yaml` and `kubectl apply -f default-backend-deploy.yaml`
- check it out: `kubectl get all --namespace=ingress -o wide`

4. Create config map for the ingress controller: 
- file: **nginx-ingress-controller-conf.yaml**

5. The Nginx controller deployment:
- image source from [quay](https://quay.io/repository/kubernetes-ingress-controller/nginx-ingress-controller)
- read the [release notes](https://github.com/kubernetes/ingress-nginx/releases)
- if you like, you can pull the docker image manually: `docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1`

6. Before we come to our nginx controller deployment, we need to set up two roles for the nginx service account: 
- ClusterRole: **nginx-ingress-clusterrole** (cluster wide definition)
- Role in the ingress namespace: **nginx-ingress-role**
- both in the file: **nginx-ingress-controll-roles.yaml**
- deploy both roles: `kubectl apply -f nginx-ingress-controll-roles.yaml`

7. The deployment is defined in the **nginx-ingress-controll-deploy.yaml** with reference to the previously deployed roles.
- `kubectl apply -f nginx-ingress-controll-deploy.yaml`

8. The nginx controller service will define all L7 upstream routing to our apps:
- file: **nginx-ingress-controll-svc.yaml**
- ingress objects for our nginx and the apps:
	- file: **ingress-nginx.yaml**
	- file: **ingress-aplikacje.yaml**
- deploy all of the above with the `kubectl apply -f <file>` command.

### for those who do not have local DNS configured:

- LoadBalancer IP: <IP> 
- on the kvm host in `/etc/hosts` put the following entry:
	- ```<IP>	app.<nazwa>.com```

9. Test the deployment from the host:
- `curl http://app.<name>.com/app1` and `curl http://app.<name>.com/app1` 
- first should return NGINX default page, the second the Apache's one.

------------
There is a shorter way to achieve the above, just go [there](kubernetes/README.md) and run two `kubectl ..` commands.
