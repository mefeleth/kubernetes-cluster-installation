# Kubernetes cluster installation using 3-node VirtualBox VMs and Kubeadm
### This guide serves as an example step-by-step setup of Kubernetes cluster that consists of 3 nodes (1 master + 2 workers) with the use of VirtualBox virtual machines and
Kubeadm installation utility.

##### 1. Create a Host-only network in VirtualBox
###### a. `File > Host Network Manager > Create`
Wait until the creation of new network adapter and click `Properties`
###### b. Disable DHCP (we will set static IP addresses later) 
###### c. In my case I'm gonna change the 3rd octet of IP address to 99, so in IPv4 address I'm gonna type 192.168.99.1. Leave the subnet mask to be /24.

##### 2. Create the master node
In my case I'm gonna name it kube-master and it's gonna be a Lubuntu LXDE 18.04 machine with 2 network interfaces - one for internet access and one for kubernetes network.
###### a. Assign 2 CPUs for the node and 4GB RAM, 10GB of disk space will be sufficient
###### b. Set the first network adapter to be a NAT with Desktop type card and forward port 22 (guest) to 2711 (host). 
* Keep a note in your head * that you will have to change the port on host in additional nodes to 2711+i, where i is represented by the consecutive worker nodes.
###### c. Set the second network adapter to be a Host-only that we created earlier.
###### d. Launch the machine and install Lubuntu.
* Note * If you will encounter an error while starting a machine, saying that there is no such Ethernet adapter, on your Windows PC go to `Control Panel > Network and Sharing Center >
Change Network Interface Card Settings > Right-click on the Ethernet assigned to VBox Host-only > Disable and Enable again`

###### 3. Install and configure necessary utilities
Login to your newly created kubernetes master machine and install and configure necessary utilities (terminator is optional):
##### a. Install necessary utilities:
```
sudo apt-get update && sudo apt-get install -y gcc make perl vim curl terminator net-tools openssh-server openssh-client
```

##### b. (Optional but recommended for the sake of laziness) Mount Guest Addition ISO and install Guest Addition (replace X with whatever version you have or complete with a tab)
```
cd /media/$USER/VBox_GAs_X.X.X
sudo ./VBoxLinuxAdditions.run
reboot
```

##### 4. Set static IP address (* Keep a note in your head * that you will have to change the addresses section to appropriate IP addresses in workers separately later):
```
sudo vim /etc/netplan/01-network-manager-all.yaml
```
The file should look like below:
```
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    <interface_name, f.i. enp0s8>:
      dhcp4: no
	  addresses:
	    - 192.168.99.2/24
	  gateway4: 192.168.99.1
```
```
sudo netplan apply
reboot
```

* Note * If you're using *buntu images below 17.04 version, this will be the correct way of setting the static IP addresses instead:
```
vim /etc/network/interfaces
```
Add lines:
```
auto <interface>
iface <interface> inet static
	address 192.168.99.2
	netmask 255.255.255.0
```

Make sure the IP address was assigned:
```
ifconfig <interace_name, f.i. enp0s8> 
```

##### 5. Set the config to let the iptables see bridged traffic
###### a. Check if the br_netfilter modules is loaded:
```
lsmod | grep br_netfilter
```
If nothing is returned, do:
```
sudo modprobe br_netfilter
```
###### b. Load momdules and configure sysctl to see bridged traffic via iptables
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```
sudo sysctl --system
```

##### 6. Add entries to `/etc/hosts`
```
sudo vim /etc/hosts
```
Under localhost add the following lines:
```
192.168.99.2	kubernetes-master
192.168.99.3	kubernetes-worker1
192.168.99.4	kubernetes-worker2
```

Optionally add loadbalancer if you want to use one:
```
192.168.99.5	kubernetes-lb
```

##### 7. Disable swap
```
sudo swapoff -a
```
```
sudo vim /etc/fstab
```
*Make sure the swap line is commented*

##### 8. (Optional) Configure DNS
```
sed -i -e 's/#DNS=/DNS=8.8.8.8/' /etc/systemd/resolved.conf
service systemd-resolved restart
```

##### 9. Install Docker
```
sudo apt-get update && sudo apt-get install -y \
  apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
sudo apt-get update && sudo apt-get install -y containerd.io docker-ce docker-ce-cli
```

Elevate yourself to root
```
sudo su
```
```
cat > /etc/docker/daemon.json <<EOF
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
```
exit
```
```
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Verify the installation
```
sudo docker ps -a
```

##### 10. Install Kubernetes utilities
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update && sudo apt-get install -y kubelet kubectl kubeadm
sudo apt-mark hold kubelet kubectl kubeadm
poweroff
```


##### 11. Clone the VM
###### a. Right-click on your kubernetes-master vm > Clone > change the name to kubernetes-worker1
###### b. Set `MAC Address Policy` to `Generate new MAC addresses for all network adapters`
###### c. Do a Full clone
###### d. Repeat for kubernetes-worker2

##### 12. Configure worker VMs
###### a. Kubernetes-worker1 > Settings > Network > Advanced options for the 1st network adapter > Port forwarding > Add > Host port 2712 to Guest port 22
###### b. Repeat the same for the kubernetes-worker2 but change the Host port to 2713.
###### c. You can decrease the RAM range to 2GB for the workers.

Launch all kubernetes VMs.


##### 13. Change hostnames, /etc/hosts and static IPs on workers
###### a. On kubernetes-worker1
```
sudo vim /etc/hostname
```
Modify the hostname to kubernetes-worker1

```
sudo vim /etc/hosts
```
Modify name next to 127.0.1.1 to kubernetes-worker1.

```
sudo vim /etc/netplan/01-network-manager-all.yaml
```
change the line:
```
	addresses:
		- 192.168.99.3/24
```
```	
sudo netplan apply
reboot
```
		
###### b. On kubernetes-worker2
```
sudo vim /etc/hostname
```
Modify the hostname to kubernetes-worker2

```
sudo vim /etc/hosts
```
Modify name next to 127.0.1.1 to kubernetes-worker1.

```
sudo vim /etc/netplan/01-network-manager-all.yaml
```
change the line:
```
	addresses:
		- 192.168.99.4/24
```
```	
sudo netplan apply
reboot
```

###### c. Verify the appropriate ip addresses are assigned to each worker:
```
ifconfig <interface>
```

###### d. Make sure the nodes can talk to each other
From master:
```
ssh kubernetes-worker1
ssh kubernetes-worker2
```
Repeat from all nodes and ssh from workers to master.


##### 14. Decide upon pod network addon
Now it's time to decide which Container Network Interface you want to use in your Kubernetes cluster, as the cluster initalization could depend on CNI options you provide.
I will be using Weave Net in this example.

##### 15. Initialize cluster with kubeadm (on master)
```
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.99.2
```

In --pod-network-cidr option I am specifying the pod network range that does not conflict with the network of the nodes.
In --apiserver-advertise-address option I am specifying the static IP address of the master, since here the kube-apiserver will be deployed.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Make a note of the kubeadm join command - copy it somewhere as we will need  this later.


##### 16. Install pod network plugin
In this case I am installing Weave Net.
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Verify the weave and core-dns pods are running:
```
kubectl get pods --all-namespaces
```

##### 16. Join workers to the network
Paste the kubeadm join returned by previous kubeadm init command in each worker node.
```
sudo kubeadm join 192.168.99.2:6443 --token <token> \
	--discovery-token-ca-cert-hash sha256:<hash>
```

##### 17. Verify the nodes joined our network
```
kubectl get nodes -o wide
```

If everything went well we should be able to see all nodes in Ready state.

Run below commands to make sure the pods are being scheduled on the nodes and there is an external internet connection:
```
kubectl run nginx --image=nginx
```
```
kubectl run nginx-lower --image=nginx:1.17
```
```
kubectl get pods -o wide
```
