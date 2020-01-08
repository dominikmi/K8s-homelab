
# Deploying test apps to K8s with access through nginx ingress controller

## Task

We would like to test an app deployment, where the app will has 2 replicas accessible via port 80 defined in the app service.
The nginx deployment will be our load balancer object with default back-end. All ingress traffic coming to our apps as app.dominik.com/app1 or ../app2 will be routed to the respective deployed app.

	- Applications namespace: aplikacja

1. First, we create two apps deployment (hello-app and a default Apache) from dockersamples/static-site and dockersamples/httpd
- file: **app-deployment.yaml**

2. Now, we create service objects for each app, exposing the app with their pod IP and port 80/tcp:
- file: **app-service.yaml**

3. Set up current context to namespace **aplikacja**: `kubectl config set-context --current --namespace=aplikacja` 
4. Deploy apps and their services: `kubectl apply -f app-deployment.yaml -f app-service.yaml`
5. Check out the deployment: `kubectl get all -o wide -n aplikacja`
6. Create ingress object for our apps: **ingress-aplikacje.yaml**

