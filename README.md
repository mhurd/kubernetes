# kubernetes

These are my notes and relevant yaml files for the setup of my Raspberry Pi cluster (4x Raspberry Pi 3 Model B's). Note this is relevant for these versions of Kubernetes and Docker:

Kubernetes: 
```text
&version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T20:55:30Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/arm"}
```

Docker:
```text
Docker version 17.11.0-ce, build 1caf76c
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
We're using a private network with all pis connected using eth0 via the switch and one master node connected to the main network via WiFi. 

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
# also add a route to keep the default 10.96.0.0/12 service network inside the private network
ip route add 10.96.0.0/12 dev eth0
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

Now on the master node use the kubeadm init command to bootstrap the cluster:
```bash
sudo kubeadm init --apiserver-advertise-address 10.0.0.1 --ignore-preflight-errors Swap
```
Note that we have to tell the setup that we are using the 10.0.0.1 interface on the master node for the api server. Also we suppress errors about running with swap enabled.

This can take several minutes, once complete it will let you know the join command to use on the worker nodes, something like the following but with different token/hash (we'll alter this slightly before using it):
```bash
kubeadm join --token d2d10c.7243e91740e722c7 10.0.0.1:6443 --discovery-token-ca-cert-hash sha256:c121856e77f33e4454efb948cb6f7e408700f744e05432969f1bc4ec5b7d7eae
```
It will also instruct you to run the following on the master node (I added the *rm* at the beginning in case you've run an init before):
```bash
rm -rf $HOME/.kube
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Now we need to prepare the kube-proxy configuration to set it up for using weave-net, run:
```bash
kubectl -n kube-system edit ds kube-proxy
```
Add in this parameter: *--cluster-cidr=10.32.0.0/12*, i.e.
```yaml
containers:
        - command:
          - kube-proxy
          - --kubeconfig=/run/kubeconfig
          - --cluster-cidr=10.32.0.0/12           <--- Added this
```
Now we need to delete the kube-proxy pods so they are recreated, repeat for each node, use *kubectl get pods --all-namespaces -o wide* to find the names assigned:
```bash
kubectl -n kube-system delete pods kube-proxy-{NODE_ID}
```
Once the kube-proxy pods have restarted with our new config we can install weave-net using the following command:
```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Now we can ssh to each of the worker node and join them using the string that the init command gave us earlier, just slightly modified as per the following:
```bash
sudo kubeadm join --token d2d10c.7243e91740e722c7 10.0.0.1:6443 --discovery-token-ca-cert-hash sha256:c121856e77f33e4454efb948cb6f7e408700f744e05432969f1bc4ec5b7d7eae --ignore-preflight-errors Swap
```
Now you can return to the master node and check for all our new nodes and relevent pods:
```bash
kubectl get pods --all-namespaces -o wide
```

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
