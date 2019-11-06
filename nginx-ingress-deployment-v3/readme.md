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
Dostęp do aplikacji z zewnątrz (z hosta w sieci lokalnej poprzez virbr0: 192.168.122.1)

Jako load balancer wykorzystamy nginx z defaultowym back-endem. Wszystko z zewnątrz co przychodzi na app.<nazwa>.com/app1 i app.<nazwa>.com/app2  ma być przekierowane na aplikację w klastrze.

- Namespace dla aplikacji: aplikacja
- Namespace dla nginx i obiektu ingress: ingress

1. Najpierw tworzymy przykładowe dwie aplikacje z dwiema replikami każda (hello-app) z dockersamples/static-site.

-> plik: app-deployment.yaml


2. Teraz tworzymy dwa serwisy dla każdej z aplikacji, które będę wystawiały aplikację na świat z adresem danego poda na porcie 80:

-> plik: app-service.yaml

3. Ustawiamy bieżący kontekst na namespace: aplikacja: kubectl config set-context --current --namespace=aplikacja 
4. Tworzymy teraz deployment aplikacji i serwisy: kubectl apply -f app-deployment.yaml -f app-service.yaml
5. Sprawdzamy w przestrzeni: aplikacja - czy jest Ok: kubectl get all -o wide -n aplikacja
6. Tworzenie namespace ingress - kubectl create namespace ingress.
7. Tworzymy teraz defaultowy backend dla nginx, analogicznie troche jak wyzej - czyli deployment i ekspozycja w sieci przez service:

-> plik: default-backend-deploy.yaml
-> plik: default-backend-svc.yaml

8. Tworzymy: kubectl apply -f default-..jeden i drugi  Działa? kubectl get all --namespace=ingress -o wide
9. Tworzymy ConfigMap dla nginx jako ingress-controllera (kontrolera ruchu wchodzącego do klastra) - mozna albo przez Annotations, albo ConfigMap, albo przez skastomizowane template-y w podmontowanym Volume (PV i PVC):

-> plik: nginx-ingress-controller-conf.yaml

10. Tworzymy deployment samego nginxa - kontrolera ruchu przychodzącego/wchodzącego do klastra:
 - image pobierzemy sobie z quaya: https://quay.io/repository/kubernetes-ingress-controller/nginx-ingress-controller (quay'a polecam - bo ma security skaner i mozna od razu podejrzeć jakiego "dziurawca" ściągamy) 
 - poczytajmy: https://github.com/kubernetes/ingress-nginx/releases
 - i na wszelki wypadek możemy ściągnąć ręcznie: docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1

11. Zanim utworzymy deployment, utworzmy role  dla samego konta usługowego (serviceaccount) nginx-a: Klastrowa - definiowana prez ClusterRole i nazwana: nginx-ingress-clusterrole, i 	w ramach przestrzeni nazw: ingres - nazwana: nginx-ingress-role

-> plik: nginx-ingress-controll-roles.yaml

12. Teraz dopiero możemy utworzyć deployment samego nginx-a, który dostanie odpowiednie uprawnienia via RBAC K8sowy:

-> plik: nginx-ingress-controll-deploy.yaml

13. A teraz czas na serwis nginx'a i reguły rutujące ruch L7 w odpowiednie miejsca:

-> plik: nginx-ingress-controll-svc.yaml

14. Obiekt ingress dla nginx-a i dla aplikacji :

-> plik: ingress-nginx.yaml
-> plik: ingress-aplikacje.yaml

WAŻNE - dla tych, którzy nie mają lokalnego DNSa pod ręką:

IP LoadBalancera: <IP> 
Na hoście kvm-emowym, w /etc/hosts wstawiamy:
<IP>	app.<nazwa>.com

i teraz curl http://app.<nazwa>.com/app1 i ../app2

pierwszy curl powinien pokazać NGINXa a drugi Apache'a,

Do pomocy w rozwiązywaniu problemów natrafiłem na taki shell-script który listuje wszystkie resourcy i sub-resourcy w grupach API:
$ _list=($(kubectl get --raw / |grep "^    \"/api"|sed 's/[",]//g')); for _api in ${_list[@]}; do _aruyo=$(kubectl get --raw ${_api} | jq .resources); if [ "x${_aruyo}" != "xnull" ]; then echo; echo "===${_api}==="; kubectl get --raw ${_api} | jq -r ".resources[].name"; fi; done

I już mamy podstawę do twórczego rozwijania własnego K8s. :)


