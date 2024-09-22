## Set up a Highly Available Kubernetes Cluster using kubeadm
## Structure
![image](https://hackmd.io/_uploads/SJZ0exApA.png)

|  ROLE | HOSTNAME     | IP | S.O |  Memory   | CPU |API-VERSION|
| ------| -----| -- | --- | -------- | --- | -------- |
|HAproxy|haproxy|10.0.1.80|Ubuntu 22.04|512M|1|  |
|MASTER    |kmaster0|10.0.1.20|Ubuntu 22.04|4G|2| 1.27.3 |
|MASTER    |kmaster1|10.0.1.21|Ubuntu 22.04|4G|2| 1.27.3 |
|WORKER    |kworker0|10.0.1.30|Ubuntu 22.04|1G|1| 1.27.3 |
|WORKER    |kworker1|10.0.1.31|Ubuntu 22.04|1G|1| 1.27.3 |



## Setup haproxy node

```bash=
sudo apt update && sudo apt install -y haproxy
```

insert config in this file **/etc/haproxy/haproxy.cfg**
```yaml
frontend kubernetes-frontend
    bind 10.0.1.80:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server kmaster1 10.0.1.20:6443 check fall 3 rise 2
    server kmaster2 10.0.1.21:6443 check fall 3 rise 2
```
Restart haproxy service
```bash=
systemctl restart haproxy
```

# Install and configure keepalived

```bash=
sudo apt intall keepalived
```
edit 
/etc/keepalived/keepalived.conf


```config=
vrrp_instance VI_1 {
    state MASTER
    interface <YOUR_INTERFACE>  # e.g., eth0
    virtual_router_id 51
    priority 101  # Higher priority for Master
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass my_secret_password
    }
    virtual_ipaddress {
        192.168.1.100  # Your desired VIP
    }
}
```



# Kubernetes setup in all kubernetes noode

Disable swap

```bash=
sudo su
# Disable swap
sudo swapoff -a; sed -i '/swap/d' /etc/fstab
```

### ContainerD Setup


check for updates in:
https://github.com/containerd/containerd/releases/
```bash=
wget https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
# Unpack that file into /usr/local/ with the command:
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
```

### RunC Setup
check for updates in:
https://github.com/opencontainers/runc/releases

```bash=
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
# Install binary
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```
### Download cni plugin
check for updates in:
https://github.com/containernetworking/plugins/releases
```bash=
wget https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-amd64-v1.4.1.tgz
# Create a new directory with:
sudo mkdir -p /opt/cni/bin
# Unpack the CNI file into our new directory with:
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.1.tgz
```
How to configure Containerd

```bash=
sudo mkdir /etc/containerd
#Create the configurations with:
containerd config default | sudo tee /etc/containerd/config.toml
# Enable SystemdCgroup with the command:
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
#Download the required systemd file with:
sudo curl -L https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /etc/systemd/system/containerd.service
#Reload the systemd daemon with:
sudo systemctl daemon-reload
#Finally, start and enable the Containerd service with:
sudo systemctl enable --now containerd
#You can verify everything is running with the command:
sudo systemctl status containerd
```
You should see output similar to this:
```systemd~
containerd.service - containerd container runtime
Loaded: loaded (/etc/systemd/system/containerd.service; enabled; vendor pre>
Active: active (running) since Wed 2022-09-21 12:17:24 UTC; 6s ago
Docs: https://containerd.io
Process: 1475 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUC>
Main PID: 1478 (containerd)
Tasks: 8
Memory: 19.4M
CPU: 257ms
CGroup: /system.slice/containerd.service
└─1478 /usr/local/bin/containerd
```


### Kubernetes Setup
**Important Read:**
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
```bash=
#update system and install components
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
# imort gpg key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# add repository 
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
install kubelet kubeadm kubectl
```bash~
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
### Download Crictl
check for updates in:
https://github.com/kubernetes-sigs/cri-tools/releases
```bash=
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.29.0/crictl-v1.29.0-linux-amd64.tar.gz
# Extract critl to /usr/local/bin
sudo tar zxvf crictl-v1.29.0-linux-amd64.tar.gz -C /usr/local/bin
# Check crictl version
crictl --version
```
### install compenents
```bash=
sudo apt-get install -y ebtables socat conntrack iptables
```
### change network process
```bash=
# Use su user
sudo su
# Change network process
echo '1' > /proc/sys/net/ipv4/ip_forward
# Load modules
sudo modprobe br_netfilter
```

### Initialize Kubernetes Cluster in node controll-plane only
```bash=
sudo kubeadm config images pull
# Init cluster
kubeadm init --cri-socket=/run/containerd/containerd.sock --control-plane-endpoint="10.0.1.84:6443" --upload-certs --apiserver-advertise-address=10.0.1.24 --pod-network-cidr=192.168.0.0/16
```


