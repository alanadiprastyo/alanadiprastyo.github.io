# Install and Configure HA Kubernetes Cluster Using Keepalived and HAproxy

---
A highly available Kubernetes cluster ensures your applications run without outages which is required for production. In this connection, there are plenty of ways for you to choose from to achieve high availability.

This tutorial demonstrates how to configure Keepalived and HAproxy for load balancing and achieve high availability. The steps are listed as below:

1. Prepare hosts.
2. Configure Keepalived and HAproxy.
3. set up a Kubernetes cluster with Kubeadm


### **Cluster Architecture**

The example cluster has three master nodes, three worker nodes, two nodes for load balancing and one virtual IP address. The virtual IP address in this example may also be called “a floating IP address”. That means in the event of node failures, the IP address can be passed between nodes allowing for failover, thus achieving high availability.

![Topologi](/img/k8s/architecture-ha-k8s-cluster.png)

Notice that in this example, Keepalived and HAproxy are not installed on any of the master nodes. Admittedly, you can do that and high availability can also be achieved. That said, configuring two specific nodes for load balancing (You can add more nodes of this kind as needed) is more secure. Only Keepalived and HAproxy will be installed on these two nodes, avoiding any potential conflicts with any Kubernetes components and services.


### **Prepare Hosts**
| IP Address 	| Hostname 	| Role 	|
|---	|---	|---	|
| 10.0.21.10 	| Bastion 	| jump server 	|
| 10.0.21.6 	| LB-Active 	| Keepalived & HAproxy 	|
| 10.0.21.7 	| LB-Passive 	| Keepalived & HAproxy 	|
| 10.0.21.11 	| control-plane-01 	| master, etcd 	|
| 10.0.21.12 	| control-plane-02 	| master, etcd 	|
| 10.0.21.13 	| control-plane-03 	| master, etcd 	|
| 10.0.21.14 	| compute-01 	| worker 	|
| 10.0.21.15 	| compute-02 	| worker 	|
| 10.0.21.5 	|  	| Virtual IP 	|


### **Configure Load Balancing**

Keepalived provides a VRPP implementation and allows you to configure Linux machines for load balancing, preventing single points of failure. HAProxy, providing reliable, high performance load balancing, works perfectly with Keepalived.

As Keepalived and HAproxy are installed on lb-active and lb-passive, if either one goes down, the virtual IP address (i.e. the floating IP address) will be automatically associated with another node so that the cluster is still functioning well, thus achieving high availability. If you want, you can add more nodes all with Keepalived and HAproxy installed for that purpose.

- Setting Firewalld
```bash
yum install policycoreutils-python-2.5-34.el7.x86_64 -y
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --permanent --add-rich-rule='rule protocol value="vrrp" accept'
sudo firewall-cmd --reload
sudo semanage port -a -t http_cache_port_t 6443 -p tcp
```

- install keepalived and haproxy
```bash
sudo yum install -y haproxy keepalived psmisc
```

#### **Setting Haproxy**
1. edit file `haproxy.cfg`
```bash
vi vi /etc/haproxy/haproxy.cfg
```
```bash
global
  log /dev/log  local0
  log /dev/log  local1 notice
  stats socket /var/lib/haproxy/stats level admin
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
  log global
  mode  http
  option  httplog
  option  dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000

frontend kubernetes
    bind 10.0.21.5:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server control-plane-01 10.0.21.11:6443 check fall 3 rise 2
    server control-plane-02 10.0.21.12:6443 check fall 3 rise 2
    server control-plane-03 10.0.21.13:6443 check fall 3 rise 2

listen stats 10.0.21.5:8080
    mode http
    stats enable
    stats uri /
    stats realm HAProxy\ Statistics
    stats auth admin:haproxy123
```
2. Enable and Start service Haproxy
```bash
sudo systemctl enable --now haproxy
sudo systemctl status haproxy
```
Note: if status haproxy `failed` don't worry because virtual ip not yet available.

3. Make sure you configure HAproxy on the other machine (lb-passive) as well.

#### **Setting Keepalived**

The host’s kernel needs to be configured to allow a process to bind to a non-local IP address. This is because non-active VRRP nodes will not have the virtual IP configured on any interfaces.

```bash
echo "net.ipv4.ip_nonlocal_bind=1" | sudo tee /etc/sysctl.d/ip_nonlocal_bind.conf
sudo sysctl --system
```
Configure Master Keepalived in lb-active
```bash
vi /etc/keepalived/keepalived.conf
```
```bash
global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from root@localhost
   smtp_server localhost
   smtp_connect_timeout 30
}

# Script used to check if HAProxy is running
vrrp_script check_haproxy {
    script "killall -0 haproxy" # check the haproxy process
    interval 2 # every 2 seconds
    weight 2 # add 2 points if OK
}

vrrp_instance VI_1 {
    state MASTER # MASTER on haproxy, BACKUP on haproxy2
    interface eth1 
    virtual_router_id 255
    priority 101 # 101 on haproxy, 100 on haproxy2
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass EhazK1Y2MBK37gZktTl1zrUUuBk
    }
    virtual_ipaddress {
        10.0.21.5
    }
    
    track_script {
        check_haproxy
    }
}
```
Configure Backup Keepalived in lb-passive
```bash
vi /etc/keepalived/keepalived.conf
```
```bash
global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from root@localhost
   smtp_server localhost
   smtp_connect_timeout 30
}

# Script used to check if HAProxy is running
vrrp_script check_haproxy {
    script "killall -0 haproxy" # check the haproxy process
    interval 2 # every 2 seconds
    weight 2 # add 2 points if OK
}

vrrp_instance VI_1 {
    state BACKUP # MASTER on haproxy, BACKUP on haproxy2
    interface eth1
    virtual_router_id 255
    priority 100 # 101 on haproxy, 100 on haproxy2
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass EhazK1Y2MBK37gZktTl1zrUUuBk
    }
    virtual_ipaddress {
        10.0.21.5
    }
    track_script {
        check_haproxy
    }
}
```
Enable and Start service keepalived
```
sudo systemctl enable --now keepalived
sudo systemctl status keepalived
```

#### **Verify High Availability**
Before you start to create your Kubernetes cluster, make sure you have tested the high availability. 


Show IP in LB-Active
```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:74:d1:d5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.6/24 brd 192.168.100.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::c7b0:608b:8afb:1985/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::c083:661:9b0e:56ec/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::c21d:97ff:ae34:37e8/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:cc:2c:31 brd ff:ff:ff:ff:ff:ff
    inet 10.0.21.6/24 brd 10.0.21.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet 10.0.21.5/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::30ac:f13d:83d3:7554/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

Show IP in LB-Passive
```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:bd:0f:fa brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.7/24 brd 192.168.100.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::c7b0:608b:8afb:1985/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::c083:661:9b0e:56ec/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::c21d:97ff:ae34:37e8/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:a6:e5:d4 brd ff:ff:ff:ff:ff:ff
    inet 10.0.21.7/24 brd 10.0.21.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::bf72:1903:a502:2367/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

as you can see, the virtual ip assign in host lb-active because it's master but if haproxy service stop or failed on lb-active then virtual ip will move to lb-passive.

### **Configure Kubernetes Cluster with CRI-O**
make sure you install both Kubernetes and CRI-O at the same version. I’m using v1.22 in this tutorial. set the OS version, and other component versions in the following script. This script is already pre-configured to install Kubernetes and CRI-O v1.22 on Centos 7 version.

Running this command on all nodes

```bash
##Update the OS
yum update -y
 
## Install additional package
yum install yum-utils net-tools bash-completion git -y
 
##Disable firewall starting from Kubernetes v1.19 onwards
systemctl disable firewalld --now
 
 
## letting ipTables see bridged networks
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
 
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
 
##
## iptables config as specified by CRI-O documentation
# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF
 
 
sudo modprobe overlay
sudo modprobe br_netfilter
 
 
# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
 
sudo sysctl --system
 
 
###
## configuring Kubernetes repositories
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
 
## Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
 
### Disable swap
swapoff -a
 
##make a backup of fstab
cp -f /etc/fstab /etc/fstab.bak
 
##Renove swap from fstab
sed -i '/swap/d' /etc/fstab
 
 
##Refresh repo list
yum repolist -y
 
## Install CRI-O binaries
##########################
#Operating system   $OS
#Centos 8   CentOS_8
#Centos 8 Stream    CentOS_8_Stream
#Centos 7   CentOS_7
 
#set OS version
OS=CentOS_7
 
#set CRI-O
VERSION=1.22
 
# Install CRI-O
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
sudo yum install cri-o -y
 
##Install Kubernetes, specify Version as CRI-O
yum install -y kubelet-1.22.0-0 kubeadm-1.22.0-0 kubectl-1.22.0-0 --disableexcludes=kubernetes
```

CRI-O uses “systemd” as the cgroup driver. Edit this file at `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf`

```
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
## The following line to be added for CRI-O 
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS $KUBELET_CGROUP_ARGS
```
reload daemon and then start CRI-O and kubelet services.
```bash
systemctl daemon-reload
systemctl enable crio --now
systemctl enable kubelet --now
```
create kubeadm config to initiate control-plane
```bash
vi kubeadm-config.yaml
```
```bash
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "10.0.21.11" #private ip control-plane01
  bindPort: 6443
---
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta2
kubernetesVersion: v1.22.0
controlPlaneEndpoint: "control-plane.k8s.example.com" # --control-plane-endpoint
networking:
  podSubnet: "10.244.0.0/16" # --pod-network-cidr
```

init cluster with kubeadm-config.yaml file
```bash
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```
join another control-plane to cluster k8s
```bash
kubeadm join control-plane.k8s.example.com:6443 --token xa1rvc.qevoeuriqbq7heil --discovery-token-ca-cert-hash sha256:1d523af1xxxxxxxxxxxxxxxx486a09b6dc1abcac88490 --control-plane --certificate-key 8f2d0ce0efxxxxxxxxxxxxxx124189
```
join another worker to cluster k8s
```bash
kubeadm join control-plane.k8s.example.com:6443 --token xa1rvc.qevoeuriqbq7heil --discovery-token-ca-cert-hash sha256:1d523af1aed490xxxxxxxxxxxxxxxx86a09b6dc1abcac88490
```
check nodes make sure all nodes status `Ready`
```
# kubectl get nodes
NAME                               STATUS   ROLES                  AGE   VERSION
compute-01.k8s.example.com         Ready    <none>                 31h   v1.22.0
compute-02.k8s.example.com         Ready    <none>                 31h   v1.22.0
control-plane-01.k8s.example.com   Ready    control-plane,master   32h   v1.22.0
control-plane-02.k8s.example.com   Ready    control-plane,master   32h   v1.22.0
control-plane-03.k8s.example.com   Ready    control-plane,master   31h   v1.22.0
```
install CNI calico
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
verify all pod running
```bash
# kubectl get pod --all-namespaces
NAMESPACE     NAME                                                       READY   STATUS    RESTARTS      AGE
kube-system   calico-kube-controllers-75f8f6cc59-dcdzc                   1/1     Running   0             32h
kube-system   calico-node-4jn6d                                          1/1     Running   0             32h
kube-system   calico-node-6987w                                          1/1     Running   0             32h
kube-system   calico-node-9crsb                                          1/1     Running   0             32h
kube-system   calico-node-9k22j                                          1/1     Running   0             32h
kube-system   calico-node-rgggc                                          1/1     Running   0             32h
kube-system   coredns-78fcd69978-vshr6                                   1/1     Running   0             32h
kube-system   coredns-78fcd69978-xrxq7                                   1/1     Running   0             32h
kube-system   etcd-control-plane-01.k8s.example.com                      1/1     Running   2             32h
kube-system   etcd-control-plane-02.k8s.example.com                      1/1     Running   0             32h
kube-system   etcd-control-plane-03.k8s.example.com                      1/1     Running   0             32h
kube-system   kube-apiserver-control-plane-01.k8s.example.com            1/1     Running   2             32h
kube-system   kube-apiserver-control-plane-02.k8s.example.com            1/1     Running   1             32h
kube-system   kube-apiserver-control-plane-03.k8s.example.com            1/1     Running   3 (32h ago)   32h
kube-system   kube-controller-manager-control-plane-01.k8s.example.com   1/1     Running   4 (32h ago)   32h
kube-system   kube-controller-manager-control-plane-02.k8s.example.com   1/1     Running   1             32h
kube-system   kube-controller-manager-control-plane-03.k8s.example.com   1/1     Running   1             32h
kube-system   kube-proxy-jr8vb                                           1/1     Running   0             32h
kube-system   kube-proxy-n7pdx                                           1/1     Running   0             32h
kube-system   kube-proxy-ndsdt                                           1/1     Running   0             32h
kube-system   kube-proxy-pnd2t                                           1/1     Running   0             32h
kube-system   kube-proxy-rw8sj                                           1/1     Running   0             32h
kube-system   kube-scheduler-control-plane-01.k8s.example.com            1/1     Running   4 (32h ago)   32h
kube-system   kube-scheduler-control-plane-02.k8s.example.com            1/1     Running   1             32h
kube-system   kube-scheduler-control-plane-03.k8s.example.com            1/1     Running   1             32h
```

Verify Dashboard Haproxy
![Dashboard Haproxy](/img/k8s/dashboard-haproxy.png)


Ref:
- https://kubesphere.io/docs/installing-on-linux/high-availability-configurations/set-up-ha-cluster-using-keepalived-haproxy/
- https://www.lisenet.com/2021/install-and-configure-a-multi-master-ha-kubernetes-cluster-with-kubeadm-haproxy-and-keepalived-on-centos-7/
- https://arabitnetwork.com/2021/02/20/install-kubernetes-with-cri-o-on-centos-7-step-by-step/