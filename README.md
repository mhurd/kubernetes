# kubernetes

These are my notes and relevant yaml files for the setup of my Raspberry Pi cluster (4x Raspberry Pi 3 Model B's). Note this is relevant for these versions of Kubernetes and Docker:

Kubernetes: 
```text
&version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T20:55:30Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/arm"}
```

Docker:
```text
Docker version 17.12.0-ce, build c97c6d6
```

## Raspberry Pi Cluster shopping list

* [Multi-Pi Stackable Raspberry Pi Case (x2)](https://www.modmypi.com/raspberry-pi/cases-183/raspberry-pi-b-plus2-and-3-cases-1122/stacking-cases-1132/multi-pi-stackable-raspberry-pi-case/?search=stackab) 
* [Raspberry Pi 3 Model B Quad Core CPU 1.2 GHz 1 GB RAM Motherboard (x4)](https://www.amazon.co.uk/gp/product/B01CD5VC92/ref=od_aui_detailpages00?ie=UTF8&psc=1)
* [SanDisk Ultra 32GB micro SDHC Memory Card + SD Adapter with A1 App Performance up to 98MB/s, Class 10, U1 - FFP (x3)](https://www.amazon.co.uk/gp/product/B073S8LQSL/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [SanDisk Ultra 64GB micro SDXC Memory Card + SD Adapter with A1 App Performance up to 100MB/s, Class 10, U1 - FFP (x1)](https://www.amazon.co.uk/gp/product/B073SB2L3C/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [Adaptare 40545 Charging Cable USB Type A Male to DC Barrel Jack (5.5 x 2.5 mm – 60 cm) (x1)](https://www.amazon.co.uk/gp/product/B01BHEAM9Q/ref=oh_aui_detailpage_o01_s00?ie=UTF8&psc=1)
* [VELCRO Brand Stick On Tape - 20 mm x 2.5 m, Black (x1)](https://www.amazon.co.uk/gp/product/B0013D8IUC/ref=oh_aui_detailpage_o02_s00?ie=UTF8&psc=1)
* [Gorilla Pi Heat-sink For Raspberry Pi 3 & Pi 2 Model B. 3Pc Set (x1 copper x2 aluminium) With Pre Installed Heat-sink Adhesive (x4)](https://www.amazon.co.uk/gp/product/B01FY9196K/ref=oh_aui_detailpage_o03_s00?ie=UTF8&psc=1)
* [NETGEAR GS305-100UKS 5-Port Gigabit Metal Ethernet Desktop/Wall-mount Switch (x1)](https://www.amazon.co.uk/gp/product/B00TV4A96Q/ref=oh_aui_detailpage_o04_s00?ie=UTF8&psc=1)
* [Anker USB Charger Power-Port 5 (40W 5-Port USB Charging Hub) Multi-Port Wall Charger (x1)](https://www.amazon.co.uk/gp/product/B00VTI8K9K/ref=oh_aui_detailpage_o04_s00?ie=UTF8&psc=1)
* [Micro USB Cable Ulreon 4 Pack Short 12 inch/0.3m High Speed Quick Charging Sync 2.4A Ultra Strong Durable Nylon Braided Cord (x1)](https://www.amazon.co.uk/gp/product/B074DVPLQP/ref=oh_aui_detailpage_o05_s00?ie=UTF8&psc=1)
* [Ethernet Cable, Rankie 5-Pack 0.3m RJ45 Cat 6 Ethernet Patch LAN Network Cable (5-Color Combo) - R1300A (x1)](https://www.amazon.co.uk/gp/product/B01J8KFTB2/ref=oh_aui_detailpage_o05_s00?ie=UTF8&psc=1)

## OS
Use the latest Raspian Stretch Lite img [from here](https://downloads.raspberrypi.org/raspbian_lite_latest)
You need to make a minor adjustment to the */boot/cmdline.txt* file to enable cgroup memory required by kubernetes, add the following option:
```bash
cgroup_memory=1
```
Lower the swappiness - rather than completely removing the swap which is recommended for Kuberenetes, we only have 1Gb of RAM to play with. Edit */etc/sysctl.conf* and add:
```bash
vm.swappiness = 1
```
"1" means it will only use swap when RAM use exceeds 99%. 
Set up any niceties such as ssh keys for hopping between the nodes:
```bash
ssh-keygen -o -a 100 -t ed25519
ssh user@host "echo '`cat ~/.ssh/id_ed25519.pub`' >> ~/.ssh/authorized_keys"
```

## Docker installation
Don't use apt-get! The commands below assume that your Raspian user is called *pi*
```bash
sudo curl -sSL https://get.docker.com | sh
sudo usermod -aG docker pi
```
## Basic networking set-up
We're using a private network with all Pis connected using eth0 via the switch and one master node connected to the main network via WiFi. 

### Set up the master node as a DHCP server
First set up a static IP address for the cluster's internal network on the master node. Edit */etc/network/interfaces.d/eth0*:
```properties
allow-hotplug eth0
iface eth0 inet static
    address 10.0.0.1
    netmask 255.255.255.0
    broadcast 10.0.0.255
    gateway 10.0.0.1
```
The master node will now be 10.0.0.1 on the internal network (reboot to set this). 
Next we need to install DHCP on this master node so it will allocate IP addresses to the worker nodes. Run:
```bash
apt-get install isc-dhcp-server
```
Then configure the DHCP as follows in */etc/dhcp/dhcpd.conf*
```properties
# Set a domain name, can basically be anything
option domain-name "magnum.home";
# Use Google DNS by default, you can substitute ISP-supplied values here
option domain-name-servers 8.8.8.8, 8.8.4.4;
# We'll use 10.0.0.X for our subnet
subnet 10.0.0.0 netmask 255.255.255.0 {
    range 10.0.0.1 10.0.0.10;
    option subnet-mask 255.255.255.0;
    option broadcast-address 10.0.0.255;
    option routers 10.0.0.1;
}
default-lease-time 600;
max-lease-time 7200;
authoritative;
```
Then restart the DHCP server with:
```bash
sudo systemctl restart dhcpd
```
Now the master node should be handing out IP addresses. You can test this by hooking up a second machine to the switch via the Ethernet. This second machine should get the address 10.0.0.2 from the DHCP server.

### Private to main network routing
The final step in setting up networking is setting up network address translation (NAT) so that the worker nodes can reach the main network. Edit */etc/rc.local* (or equivilant) and add iptables rules for forwarding eth0 to wlan0 and back.
```bash
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
```
At this point the remaining worker nodes can be plugged in and get their allocated IP addresses, you can se that they are allocated the correct IP addresses by checking */var/lib/dhcp/dhcpd.leases* on the master node.

Now on each node modify the */etc/hosts* file and enter mapping for all the nodes against their allocated IP addresses.

## Installing Kubernetes
Add the encryption key for the packages:
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```
Then add the repository to the system:
```bash
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list
```
Finally, update and install the Kubernetes tools. This will also update all packages on the system.
```bash
apt-get update
apt-get upgrade
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```
Repeat this for all the nodes.

## Setting up the Kubernetes Cluster
Note, to undo this initialisation so you can run init again (I did this *many* times whilst trying to get all this working) run the following on the master node and on each of the joined worker nodes:
```bash
sudo kubeadm reset
```
On the master node (the one running DHCP and connected to the main network via WiFi) we need to make some changes to the default systemd scripts as we are going to use weave-net for pod networking and we also want to suppress error about running with swap enabled.
```bash
sudo vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
Add *--fail-swap-on=false* to the args in ExecStart and also remove $KUBELET_NETWORK_ARGS from ExecStart. Repeat this on all the nodes.

Now on the master node use the kubeadm init command to bootstrap the cluster, only add the *--pod-network-cidr* option if you are going to use Flannel pod networking, *not* for weave-net:
```bash
sudo kubeadm init --apiserver-advertise-address 10.0.0.1 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors Swap
```
Note that we have to tell the setup that we are using the 10.0.0.1 interface on the master node for the api server. Also we suppress errors about running with swap enabled.

This can take several minutes, once complete it will let you know the join command to use on the worker nodes, something like the following but with different token/hash (we'll alter this slightly before using it):
```bash
kubeadm join --token 3d7f5a.5b28483cb18857ef 10.0.0.1:6443 --discovery-token-ca-cert-hash sha256:78da1d32aac1bce32be2222cfb0ccc0c37d52399df18df7daa9425c1d2df1d91
```
It will also instruct you to run the following on the master node (I added the *rm* at the beginning in case you've run an init before):
```bash
rm -rf $HOME/.kube
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### Prepare for Pod network
This is required to be set for various networking solutions:
```bash
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

### Setting up Flannel for pod networking
Now we need to modify the flannel config to account for the required arm architecture and to force the correct port for use in the networking (flannel uses the first interface - for master this is the external wlan0 interface which is incorrect). Download the flannel yaml locally:
```bash
curl https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml >> kube-flannel.yml
```
Replace any occurances of *amd64* with *arm*. Also add the required interface option the the container setup: *--iface=eth0*, i.e.
```yaml
containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.9.1-arm
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth0                      <--- added this
```
First add the RBAC config:
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```
Now we can install the daemon set using the modified file:
```bash
kubectl apply -f kube-flannel.yml
```
Now we need to set the docker bridge interface to use the correct Flannel subnet, run the following on each of the nodes:
```bash
cat /run/flannel/subnet.env
```
This will output some Flannel properties we can use:
```text
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```
Now create a docker config file in the following location and, using the *FLANNEL_SUBNET* and *FLANNEL_MTU* details from flannel, set the *bip* and *mtu* properties:
```bash
sudo vim /etc/docker/daemon.json
```
For the properties shown above we'd put the following in the *daemon.json* file:
```text
{
    "bip": "10.244.0.1/24",
    "mtu": 1450
}
```
Repeat for all the nodes, you should probably reboot all the nodes now to get everything restarted.

### Setting up weave-net for pod networking
**NOTE could not get pod networking working with weave-net, moved to Flannel in the end. No errors in the weave-net logs, but trying a nslookup via a busybox container in the cluster failed (seemed like it couldn't get to the kube-dns service at 10.96.0.10)**

Or as an alternative to Flannel you can install weave-net instead. 
We need to prepare the kube-proxy configuration to set it up for using weave-net, run:
```bash
kubectl -n kube-system edit ds kube-proxy
```
Add in this parameter: --cluster-cidr=10.32.0.0/12, i.e.
```bash
containers:
        - command:
          - kube-proxy
          - --kubeconfig=/run/kubeconfig
          - --cluster-cidr=10.32.0.0/12           <--- Added this
```
Then apply the networking.
```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
### Joining the worker nodes to the network
Now we can ssh to each of the worker node and join them using the string that the init command gave us earlier, just slightly modified as per the following:
```bash
sudo kubeadm join --token 487b29.d6b61fd342f2143b 10.0.0.1:6443 --discovery-token-ca-cert-hash sha256:944372bf8e59936302783cf870f1e1f29509535522f0cd6fb96a2aad4a52428c --ignore-preflight-errors Swap
```
Now you can return to the master node and check for all our new nodes and relevent pods:
```bash
kubectl get pods --all-namespaces -o wide
```
Which you should return something like this:
```bash
pi@magnum:~ $ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE       IP           NODE
default       postgres-79c57766c5-tv249               1/1       Running   2          2d        10.244.1.2   bresson
kube-system   etcd-magnum                             1/1       Running   9          6d        10.0.0.1     magnum
kube-system   kube-apiserver-magnum                   1/1       Running   5          6d        10.0.0.1     magnum
kube-system   kube-controller-manager-magnum          1/1       Running   4          6d        10.0.0.1     magnum
kube-system   kube-dns-7b6ff86f69-5mgjr               3/3       Running   9          6d        10.244.3.2   parr
kube-system   kube-flannel-ds-dz9zf                   1/1       Running   4          6d        10.0.0.5     capa
kube-system   kube-flannel-ds-k5652                   1/1       Running   4          6d        10.0.0.3     bresson
kube-system   kube-flannel-ds-vxxz4                   1/1       Running   6          6d        10.0.0.1     magnum
kube-system   kube-flannel-ds-wkzp6                   1/1       Running   1          6d        10.0.0.6     parr
kube-system   kube-proxy-65bmw                        1/1       Running   4          6d        10.0.0.3     bresson
kube-system   kube-proxy-l22td                        1/1       Running   5          6d        10.0.0.1     magnum
kube-system   kube-proxy-rbhgw                        1/1       Running   6          6d        10.0.0.5     capa
kube-system   kube-proxy-x2ttc                        1/1       Running   3          6d        10.0.0.6     parr
kube-system   kube-scheduler-magnum                   1/1       Running   9          6d        10.0.0.1     magnum
kube-system   kubernetes-dashboard-588b7f599d-7wlpl   1/1       Running   2          6d        10.244.2.2   capa
```

## Kubernetes dashboard
The default kubernetes dashboard needs some modification to set it up for arm. Download the yaml file locally to edit:
```bash
curl https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml >> kubernetes-dashboard.yaml
```
Now replace any instances of *amd64* with *arm* and apply. I've not looked into authentication in detail yet, for now you can elevate the dashboard user's previlages (not recommended in a serious environment), see: https://github.com/mhurd/kubernetes/blob/master/kubernetes-dashboard-002-admin.yaml, apply this using *kubectl*.

Next to access the dashboard you'll need to ssh tunnel to the master node and map the *8001* port to localhost:
```bash
ssh -L 8001:localhost:8001 pi@master
```
Once on the master node you can run the kubectl proxy:
```bash
kubectl proxy
```
You can now access the dashboard from the machine your ssh-ing from at the following URL:
```text
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default
```
To authenticate you'll need a bearer token for the dashboard user (with the elevated permissions), to find this run (with the correct dashboard identifier for your system):
```bash
kubectl -n kube-system describe secret kubernetes-dashboard-token-gbhln
```
You should now be able to log in using this token.

## Useful commands
1. Found that Raspian doesn't start the ssh server on start-up, use this to fix that:
```bash
sudo systemctl enable ssh.socket
```
2. Show kubelet start-up details
```bash
journalctl -xeu kubelet
```
3. Install the *weave* binary to get access to some functions to help investigate weave-net issues
```bash
curl -sSL -o /usr/local/bin/weave https://github.com/weaveworks/weave/releases/download/latest_release/weave && chmod +x /usr/local/bin/weave
```
4. Get information on all pods in the cluster
```bash
kubectl get pods -a -o wide --all-namespaces
```
5. Removing a deployment
```bash
kubectl --namespace kube-system delete deployment kubernetes-dashboard
```
6. Looking at the kube-dns logs
```bash
kubectl logs --namespace=kube-system $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name) -c kubedns
```
7. Describe the services running in the default namespace
```bash
kubectl describe svc
```
8. Using busybox (see yaml file in the repo) to check DNS
```bash
kubectl exec busybox -- nslookup postgres
```
9. Using shell in busybox
```bash
kubectl exec -ti busybox -- /bin/sh
```
10. Install a yaml config file
```bash
kubectl apply -f my_file.yaml
```
11. Allowing pods to be scheduled on master:
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```
