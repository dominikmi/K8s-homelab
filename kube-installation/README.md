# Install and configure a K8s lab.

OS: Fedora 31, 16GB RAM, 450GB SSD.
virtualization: KVM, libvirt

**Note:** This short tutorial will likely get you a nicely running home/lab K8s environment. However, any production deployment will require a bit more work. For production efforts please refer directly to the Kubernetes documentation.

### 1. Install and configure KVM, libvirtd
There's not much to do as the Fedora 31 comes along pretty well with KVM already configured and preinstalled with the system. Basically ready to go. If you don't like your default disc layout Fedora installer made for you during the installation process, you can move the /var/lib/libvirt/images folder to /home and created a symlink in place of the original images store. Please use the "Virtual Machine Manager" installed by default on Fedora 31 to create the VMs - 1 CPU, 3GB RAM for master, 1 CPU 1,5GB RAM for workers is enough for the beginning. 

### 2. Activate sshd on the system.
```
$ sudo systemctl enable sshd
$ sudo systemctl daemon-reload
$ sudo systemctl start sshd
```
### 3. Check libvirtd:

- `$ sudo lsmod |grep kvm`
- `$ sudo systemctl status libvirtd`
- `$ sudo systemctl enable libvirtd`
- `$ sudo systemctl daemon-reload`
- `$ sudo systemctl start libvirtd`
- `$ ip a`

### 4. Download Linux server image you like and start creating the virtual network.

http://releases.ubuntu.com/16.04/ - to me, Ubuntu 16.04 LTS has proven most seamless K8s deployment.

### 5. Configure the network in */etc/network/interfaces* with the following:
```
# The primary network interface
auto ens3
iface ens3 inet static
	address 192.168.122.100
	netmask 255.255.255.0
	network 192.168.122.0
	broadcast 192.168.122.255
	gateway 192.168.122.1
	# dns-* options are implemented by the resolvconf package, if installed
	dns-nameservers 192.168.122.1
	dns-search home
```

Update your KVM host /etc/hosts i.e.
```
192.168.122.100	kmaster.home	kmaster
192.168.122.101	kworker1.home	kworker1
192.168.122.102	kworker2.home	kworker2
192.168.122.103	kworker3.home	kworker3
192.168.122.1	kdns.home	kdns
```

### 6. Install prerequisites of the K8s on master and worker1-2

- `$ sudo swapoff -a`
- `$ sudo vi /etc/fstab` and hash out the swap line
- `$ sudo apt-get install -y docker.io`
- `$ sudo apt-get install -y apt-transport-https`
- `$ sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -`
- `$ sudo touch /etc/apt/sources.list.d/kubernetes.list `
- `$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list`
- `$ sudo apt-get update`

And here, you can first run this command to pick the version you like (don't try the latest one, I mean it): 
`curl -s https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep Version | awk '{print $2}'`
As of now, I'd go with 1.16.4.

- `$ sudo apt-get install -y kubelet=<version> kubeadm=<version> kubectl=<version>` 

Then I'd hold up with upgrading the trio by apt-get with the following command: `sudo apt-mark hold kubelet kubeadm kubectl`.

Set cgroupfs for docker to systemd (per [this discussion](https://github.com/kubernetes/kubeadm/issues/1394))
```
# cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Using "Virtual Machine Manager" clone the preconfigured master instance to as many worker instances as you can handle (2-3, with 1CPU and 1.5GB RAM).
On each instance change the hostname in */etc/hostname* and */etc/hosts* and replace the master instance's IP with appropriate one in */etc/network/interfaces*

On kmaster run: `sudo kubeadm init --apiserver-advertise-address="192.168.122.100" --pod-network-cidr="10.32.0.0/16" --service-cidr="10.100.0.0/16"`
After a moment, the following output showed up:
```
I0109 21:10:13.913900    4816 version.go:251] remote version is much newer: v1.17.0; falling back to: stable-1.16
[init] Using Kubernetes version: v1.16.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kmaster kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.100.0.1 192.168.122.100]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kmaster localhost] and IPs [192.168.122.100 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kmaster localhost] and IPs [192.168.122.100 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 38.002628 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.16" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node kmaster as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kmaster as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: o2zpek.6bcezlnhx3pioxls
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.122.100:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<sha256 string>
```

Once it's all done, configure your local user access to kubectl on kmaster:
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Also, you may copy the *$HOME/.kube* to your home directory on the host system. Once you install the *kubectl* package on your host (i.e. using snap service on Fedora), you will be able to do all your work with the kubes remotely.

Install CNI network pod on kmaster to the whole cluster:
 `$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

Output:
```
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```

Then you can join any number of worker nodes by running the following on each as root (any previous setup needs to be erased with `kubeadm reset` prior to issuing the join command:

`$sudo kubeadm join 192.168.122.100:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<sha256 string>`

If everything went well, you should be getting the following output:
```
$ kubectl get nodes -o wide
NAME       STATUS   ROLES    AGE     VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kmaster    Ready    master   10m     v1.16.4   192.168.122.100   <none>        Ubuntu 16.04.6 LTS   4.4.0-171-generic   docker://18.9.7
kworker1   Ready    <none>   7m28s   v1.16.4   192.168.122.101   <none>        Ubuntu 16.04.6 LTS   4.4.0-171-generic   docker://18.9.7
kworker2   Ready    <none>   7m10s   v1.16.4   192.168.122.102   <none>        Ubuntu 16.04.6 LTS   4.4.0-171-generic   docker://18.9.7
kworker3   Ready    <none>   6m59s   v1.16.4   192.168.122.103   <none>        Ubuntu 16.04.6 LTS   4.4.0-171-generic   docker://18.9.7
```

## Next steps
The next steps can be accomplished straight by kubes yaml files or helm3 charts. 

- [Install Longhorn to provide storage persistency](../longhorn/README.md)
- [Install kubes monitoring - Prometheus & Grafana](../monitoring/README.md)
- [Install resource usage metrics (metrics server)](../metrics-server/README.md)
- [Define roles and add unix users to Kubes](../adding-roles-to-users/README.md)
- [Nginx ingress controller deployment](../nginx-ingress/README.md)
- (WIP) Install Helm 3 & helm repo on Github

