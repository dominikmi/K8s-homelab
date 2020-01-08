# Install and configure a K8s lab.

System: Fedora 31, 16GB RAM, 450GB SSD.
virtualization: KVM, libvirt
=======================================

Note: This short tutorial will likely get you a nicely running home/lab K8s environment. However, any production deployment will require a bit more work. For production efforts please refer directly to the Kubernetes documentation.
=======================================

### 1. Install and configure KVM, libvirtd
There's not much to do as the Fedora 31 comes along pretty well with KVM already configured and preinstalled with the system. Basically ready to go. If you don't like your default disc layout Fedora installer made for you during the installation process, you can move the /var/lib/libvirt/images folder to /home and created a symlink in place of the original images store.

### 2. Activate sshd on the system.
```
$ sudo systemctl enable sshd
$ sudo systemctl daemon-reload
$ sudo systemctl start sshd
```
### 3. Check libvirtd:

- `$ sudo lsmod |grep kvm`
- `$ sduo systemctl status libvirtd`
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
- `$ sudo apt-get install -y kubelet kubeadm kubectl` 

vi **/etc/systemd/system/kubelet.service.d/10-kubeadm.conf**
add line at the end of the "Environment.." list: `Environment=”cgroup-driver=systemd/cgroup-driver=cgroupfs”`

Using "Virtual Machine Manager" clone the preconfigured master instance to as many worker instances as you can handle (2-3, with 1CPU and 1.5GB RAM).
On each instance change the hostname in */etc/hostname* and */etc/hosts* and replace the master instance's IP with appropriate one in */etc/network/interfaces*

On kmaster run: `sudo kubeadm init --apiserver-advertise-address="192.168.122.100" --pod-network-cidr="10.32.0.0/16" --service-cidr="10.100.0.0/16"`

Once it's all done, configure your local user access to kubectl on kmaster:
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install CNI network pod on kmaster to the whole cluster:
 `$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

Then you can join any number of worker nodes by running the following on each as root:

`$sudo kubeadm join 192.168.122.100:6443 --token tv7jsa.z2jhbyy6hl4uudo1 \
    --discovery-token-ca-cert-hash <the hash from the printout after successful kube deployment on master>`

