# Another deployment variant

The two yamls taken from:
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.26.2/deploy/static/mandatory.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.26.2/deploy/static/provider/baremetal/service-nodeport.yaml
```

Further customization required to get TLS certs in place and enable HTTPS.


