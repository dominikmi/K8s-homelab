Trzecie podejście do deploymentu nginx jako lokalnego LoadBalancera dla prywatnego klastra K8s on-premise.
----------------------------------------------------------------------------------------------------------

(Żadnego chmurowego LB nie ma.)
-----------------------------

Klaster: kmaster + 2 workery:

kmaster:  192.168.122.100
kworker1: 192.168.122.101
kworker2: 192.168.122.102

sieć usługowa K8s: 10.32.0.0/12
----------------------------

Zadanie:
-------
Aplikacja z dwoma replikami na worker1 i worker2, port 80 udostępniony przez zdefiniowany service.
Dostęp do aplikacji z zewnątrz (z hosta w sieci 172.16.20.0/24 poprzez virbr0: 192.168.122.1

Jako load balancer wykorzystamy nginx z defaultowym back-endem. Wszystko z zewnątrz co przychodzi na app.dominik.com/app1 i app.dominik.com/app2  ma być przekierowane na aplikację w klastrze.

- Namespace dla aplikacji: aplikacja
- Namespace dla nginx i obiektu ingress: ingress

1. Najpierw tworzymy przykładowe dwie aplikacje z dwiema replikami każda (hello-app) z dockersamples/static-site.
---

app-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: aplikacja1
  namespace: aplikacja
spec:
  selector:
    matchLabels:
      app: app1
  replicas: 2
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: dockersamples/static-site
        env:
        - name: Dominik
          value: app1
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aplikacja2
  namespace: aplikacja
spec:
  selector:
    matchLabels:
      app: app2
  replicas: 2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: dockersamples/static-site
        env:
        - name: Dominik
          value: app2
        ports:
        - containerPort: 80
 

2. Teraz tworzymy dwa serwisy dla każdej z aplikacji, które będę wystawiały aplikację na świat z adresem danego poda na porcie 80:
---
app-service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: serwis-aplikacji1
  namespace: aplikacja
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: aplikacja1
---
apiVersion: v1
kind: Service
metadata:
  name: serwis-aplikacji2
  namespace: aplikacja
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: aplikacja2


3. Ustawiamy bieżący kontekst na namespace: aplikacja: kubectl config set-context --current --namespace=aplikacja 
---
4. Tworzymy teraz deployment aplikacji i serwisy: kubectl apply -f app-deployment.yaml -f app-service.yaml
---
4. Sprawdzamy w przestrzeni: aplikacja - czy jest Ok:
---
hobbes@kmaster:~/ingress-deployment/V3-trzecie-podejscie$ kubectl get all -o wide
NAME                              READY   STATUS    RESTARTS   AGE    IP          NODE       NOMINATED NODE   READINESS GATES
pod/aplikacja1-6c84c94ff9-8v822   1/1     Running   0          100s   10.44.0.1   kworker1   <none>           <none>
pod/aplikacja1-6c84c94ff9-9gzfl   1/1     Running   0          100s   10.36.0.2   kworker2   <none>           <none>
pod/aplikacja2-5448d84f86-6s2cq   1/1     Running   0          71s    10.36.0.3   kworker2   <none>           <none>
pod/aplikacja2-5448d84f86-k8wh4   1/1     Running   0          71s    10.44.0.2   kworker1   <none>           <none>

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                      SELECTOR
deployment.apps/aplikacja1   2/2     2            2           100s   app1         dockersamples/static-site   app=app1
deployment.apps/aplikacja2   2/2     2            2           71s    app2         dockersamples/static-site   app=app2

NAME                                    DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES                      SELECTOR
replicaset.apps/aplikacja1-6c84c94ff9   2         2         2       100s   app1         dockersamples/static-site   app=app1,pod-template-hash=6c84c94ff9
replicaset.apps/aplikacja2-5448d84f86   2         2         2       71s    app2         dockersamples/static-site   app=app2,pod-template-hash=5448d84f86

5. i dla serwisów:
---
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/serwis-aplikacji1   ClusterIP   10.96.44.120     <none>        80/TCP    10s   app=aplikacja1
service/serwis-aplikacji2   ClusterIP   10.104.238.185   <none>        80/TCP    10s   app=aplikacja2

6. Tworzenie namespace ingress - kubectl create namespace ingress.
---
7. Tworzymy teraz defaultowy backend dla nginx, analogicznie troche jak wyzej - czyli deployment i ekspozycja w sieci przez service:
---
default-backend-deploy.yaml:
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-backend
  namespace: ingress
spec:
  selector:
    matchLabels:
      app: default-backend
  replicas: 2
  template:
    metadata:
      labels:
        app: default-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-backend
        image: gcr.io/google_containers/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi


default-backend-svc.yaml:
---
apiVersion: v1
kind: Service
metadata:
  name: default-backend
  namespace: ingress
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: default-backend


8. Tworzymy: kubectl apply -f default-..  Działa?
---
hobbes@kmaster:~/ingress-deployment/V3-trzecie-podejscie$ kubectl get all --namespace=ingress -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
pod/default-backend-58cd5ccbc4-2whc5   1/1     Running   0          36s   10.36.0.4   kworker2   <none>           <none>
pod/default-backend-58cd5ccbc4-76lh5   1/1     Running   0          36s   10.44.0.3   kworker1   <none>           <none>

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/default-backend   ClusterIP   10.110.194.217   <none>        80/TCP    27s   app=default-backend

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS        IMAGES                                        SELECTOR
deployment.apps/default-backend   2/2     2            2           36s   default-backend   gcr.io/google_containers/defaultbackend:1.0   app=default-backend

NAME                                         DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES                                        SELECTOR
replicaset.apps/default-backend-58cd5ccbc4   2         2         2       36s   default-backend   gcr.io/google_containers/defaultbackend:1.0   app=default-backend,pod-template-hash=58cd5ccbc4

9. Tworzymy ConfigMap dla nginx jako ingress-controllera (kontrolera ruchu wchodzącego do klastra) - mozna albo przez Annotations, albo ConfigMap, albo przez skastomizowane template-y w podmontowanym Volume (PV i PVC):

nginx-ingress-controller-conf.yaml:
---------------
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-controller-conf
  namespace: ingress
  labels:
    app: nginx-ingress-lb
data:
  proxy-request-buffering: "off"
  proxy-buffering: "off"
  enable-vts-status: "on"

10. Tworzymy deployment samego nginxa - kontrolera ruchu przychodzącego/wchodzącego do klastra:
 image pobierzemy sobie z quaya: https://quay.io/repository/kubernetes-ingress-controller/nginx-ingress-controller (quay'a polecam - bo ma security skaner i mozna od razu podejrzeć jakiego "dziurawca" ściągamy) 
 https://github.com/kubernetes/ingress-nginx/releases
 # docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1

ale najpierw Role dla samego nginx-a: Klastrowa - definiowana prez ClusterRole i nazwana: nginx-ingress-clusterrole, i 	w ramach przestrzeni nazw: ingres - nazwana: nginx-ingress-role

nginx-ingress-controll-roles.yaml:
-----------

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx
  namespace: ingress

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  - secrets
  - namespacese
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - configmaps
  resourceNames:
  - "ingress-controller-leader-nginx"
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - get
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - get
  - update
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nginx-ingress-clusterrole
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - nodes
  - pods
  - secrets
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - update
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - extensions
  resources:
  - ingresses/status
  verbs:
  - update

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-binding
  namespace: ingress
subjects:
- kind: ServiceAccount
  name: nginx
  apiGroup: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role # binding do roli nginx-ingress-role w namespace ingress

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nginx-ingress-clusterrole-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
- kind: ServiceAccount
  name: nginx
  namespace: ingress


i komendą: kubectl create -f nginx-ingress-controll-roles.yaml -n ingress - aplikujemy definicję ról, uprawnień i Binding konta serwisowego dla tych ról

serviceaccount/nginx created
role.rbac.authorization.k8s.io/nginx-ingress-role created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole unchanged
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-binding created

11. Teraz dopiero możemy utworzyć deployment samego nginx-a, który dostanie odpowiednie uprawnienia via RBAC K8sowy:

nginx-ingress-controll-deploy.yaml:
-------

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress
spec:
  selector:
    matchLabels:
      app: nginx-ingress-lb
  replicas: 1
  revisionHistoryLimit: 3
  template:
    metadata:
      labels:
        app: nginx-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      serviceAccount: nginx
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1
          imagePullPolicy: Always
          readinessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
          livenessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 5
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-backend
            - --configmap=$(POD_NAMESPACE)/nginx-ingress-controller-conf
            - --v=2
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - containerPort: 80
            - containerPort: 8080

hobbes@kmaster:~/ingress-deployment/V3-trzecie-podejscie$ kubectl create -f nginx-ingress-controll-deploy.yaml -n ingress
deployment.apps/nginx-ingress-controller created

Sprawdzamy:

hobbes@kmaster:~/ingress-deployment/V3-trzecie-podejscie$ kubectl get all -n ingress
NAME                                            READY   STATUS    RESTARTS   AGE
pod/default-backend-58cd5ccbc4-2whc5            1/1     Running   0          135m
pod/default-backend-58cd5ccbc4-76lh5            1/1     Running   0          135m
pod/nginx-ingress-controller-68784cd9d9-4gld7   1/1     Running   0          18s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/default-backend   ClusterIP   10.110.194.217   <none>        80/TCP    135m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/default-backend            2/2     2            2           135m
deployment.apps/nginx-ingress-controller   1/1     1            1           18s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/default-backend-58cd5ccbc4            2         2         2       135m
replicaset.apps/nginx-ingress-controller-68784cd9d9   1         1         1       18s

13. A teraz czas na serwis nginx'a i reguły rutujące ruch L7 w odpowiednie miejsca:

nginx-ingress-controll-svc.yaml:
------

apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  namespace: ingress
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 31000
      name: http # do app1 i app2
    - port: 8080
      nodePort: 32000
      name: http-mgmt # do healthz i statusu
  selector:
    app: nginx-ingress-lb

kubectl create -f nginx-ingress-controll-svc.yaml

pojawił się:
service/nginx-ingress     NodePort    10.101.29.137    <none>        80:31000/TCP,8080:32000/TCP   51s

hobbes@kmaster:~/ingress-deployment/V3-trzecie-podejscie$ kubectl get all -n ingress
NAME                                            READY   STATUS    RESTARTS   AGE
pod/default-backend-58cd5ccbc4-2whc5            1/1     Running   0          160m
pod/default-backend-58cd5ccbc4-76lh5            1/1     Running   0          160m
pod/nginx-ingress-controller-68784cd9d9-4gld7   1/1     Running   0          25m

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
service/default-backend   ClusterIP   10.110.194.217   <none>        80/TCP                         159m
service/nginx-ingress     NodePort    10.101.29.137    <none>        80:31000/TCP,8080:32000/TCP    51s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/default-backend            2/2     2            2           160m
deployment.apps/nginx-ingress-controller   1/1     1            1           25m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/default-backend-58cd5ccbc4            2         2         2       160m
replicaset.apps/nginx-ingress-controller-68784cd9d9   1         1         1       25m

--- reguły rutujące (obiekt: ingress):

ingress-nginx.yaml:
------
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: ingress
spec:
  rules:
  - host: app.dominik.com
    http:
      paths:
      - backend:
          serviceName: default-backend
          servicePort: 8080
        path: /status

ingress-aplikacje.yaml:
------
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: app-ingress
  namespace: aplikacja
spec:
  rules:
  - host: app.dominik.com
    http:
      paths:
      - backend:
          serviceName: serwis-aplikacji1
          servicePort: 80
        path: /app1
      - backend:
          serviceName: serwis-aplikacji2
          servicePort: 80
        path: /app2


 i tworzymy: kubectl create -f ingress-nginx.yaml -n ingress
             kubectl create -f ingress-aplikacje.yaml -n aplikacja


hobbes@kmaster:~/ingress-deployment/V3-trzecie-podejscie$ kubectl get ingresses --all-namespaces
NAMESPACE   NAME            HOSTS             ADDRESS   PORTS   AGE
aplikacja   app-ingress     app.dominik.com             80      107s
ingress     nginx-ingress   app.dominik.com             80      119s

============
IP LoadBalancera: 10.44.0.5
na hoście kvm-emowym, w /etc/hosts wstawiamy:
10.44.0.5 app.dominik.com

i teraz curl http://app.dominik.com/status
../app1 i ../app2

Do pomocy w rozwiązywaniu problemów natrafiłem na taki shell-script który listuje wszystkie resourcy i sub-resourcy w grupach API:
$ _list=($(kubectl get --raw / |grep "^    \"/api"|sed 's/[",]//g')); for _api in ${_list[@]}; do _aruyo=$(kubectl get --raw ${_api} | jq .resources); if [ "x${_aruyo}" != "xnull" ]; then echo; echo "===${_api}==="; kubectl get --raw ${_api} | jq -r ".resources[].name"; fi; done


