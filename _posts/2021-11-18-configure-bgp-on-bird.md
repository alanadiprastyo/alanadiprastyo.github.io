# Konfigurasi BGP Peer Calico

---

Catatan: Tulisan ini hanya untuk pribadi jadi mohon maaf jika kurang rapih dan sistematis

---

Project Calico merupakan software open sources networking untuk container, virtual machine dan native host workload. Calico juga support terhadap banyak platform diantaranya adalah Kubernetes, Openshift, Mirantis Kuberetes Engine, Openstack dan Baremetal 

Pada tulisan kali ini kita akan fokus integrasi BGP Peer pada Calico ke BGP Instance mengunakan Bird

BIRD (Bird Internet Routing Daemon) merupakan sebuah project yang berfungsi untuk menerapkan dinamis routing protokol pada Linux, FreeBSD dan lain-lain. Berikut beberapa jenis protokol routing yang disupport oleh Bird
- Both IPv4 and IPv6
- Multiple routing tables
- BGP
- RIP
- OSPF
- BFD
- Babel
- Static routes
- IPv6 Router Advertisements
- Inter-table protocol
- Command-line interface (using the `birdc' client; to get some help, just press `?')
- Powerful language for route filtering
- Linux, FreeBSD, NetBSD, OpenBSD ports

Topologi pada tulisan ini kira2 seperti berikut
![Topologi](/img/k8s/calico-peer-bgp.svg)

dimana detail networknya sebagai berikut
- Network BGP Instance (192.168.10.205) dengan ASN 64516
- Network K8s (master 192.168.10.11, worker 192.168.10.12)
- network kubernetes yang akan di advertise adalah (10.49.0.0/16, 10.244.150.192/26 dan 10.244.108.0/26)

### Install Bird pada Centos 7
```
yum install epel-release -y
yum install bird2
```
kemudian setting BGP pada bird
```
vi /etc/bird.conf
```
```
# This is a basic configuration file, which contains boilerplate options and
# some basic examples. It allows the BIRD daemon to start but will not cause
# anything else to happen.
#
# Please refer to the BIRD User's Guide documentation, which is also available
# online at http://bird.network.cz/ in HTML format, for more information on
# configuring BIRD and adding routing protocols.

# Configure logging
log syslog all;
# log "/var/log/bird.log" { debug, trace, info, remote, warning, error, auth, fatal, bug };

# Set router ID. It is a unique identification of your router, usually one of
# IPv4 addresses of the router. It is recommended to configure it explicitly.
 router id 192.168.10.205;

# Turn on global debugging of all protocols (all messages or just selected classes)
# debug protocols all;
# debug protocols { events, states };

# Turn on internal watchdog
# watchdog warning 5 s;
# watchdog timeout 30 s;

# You can define your own constants
# define my_asn = 65000;
# define my_addr = 198.51.100.1;

# Tables master4 and master6 are defined by default
# ipv4 table master4;
# ipv6 table master6;

# Define more tables, e.g. for policy routing or as MRIB
# ipv4 table mrib4;
# ipv6 table mrib6;

# The Device protocol is not a real routing protocol. It does not generate any
# routes and it only serves as a module for getting information about network
# interfaces from the kernel. It is necessary in almost any configuration.
protocol device {
}

# The direct protocol is not a real routing protocol. It automatically generates
# direct routes to all network interfaces. Can exist in as many instances as you
# wish if you want to populate multiple routing tables with direct routes.
protocol direct {
	disabled;		# Disable by default
	ipv4;			# Connect to default IPv4 table
	ipv6;			# ... and to default IPv6 table
}

# The Kernel protocol is not a real routing protocol. Instead of communicating
# with other routers in the network, it performs synchronization of BIRD
# routing tables with the OS kernel. One instance per table.
protocol kernel {
	ipv4 {			# Connect protocol to IPv4 table by channel
#	      table master4;	# Default IPv4 table is master4
#	      import all;	# Import to table, default is import all
	      export all;	# Export to protocol. default is export none
	};
#	learn;			# Learn alien routes from the kernel
#	kernel table 10;	# Kernel table to synchronize with (default: main)
}

# Another instance for IPv6, skipping default options
protocol kernel {
	ipv6 { export all; };
}

# Static routes (Again, there can be multiple instances, for different address
# families and to disable/enable various groups of static routes on the fly).
protocol static {
	ipv4;			# Again, IPv4 channel with default options

#	route 0.0.0.0/0 via 198.51.100.10;
#	route 192.0.2.0/24 blackhole;
#	route 10.0.0.0/8 unreachable;
#	route 10.2.0.0/24 via "eth0";
#	# Static routes can be defined with optional attributes
#	route 10.1.1.0/24 via 198.51.100.3 { rip_metric = 3; };
#	route 10.1.2.0/24 via 198.51.100.3 { ospf_metric1 = 100; };
#	route 10.1.3.0/24 via 198.51.100.4 { ospf_metric2 = 100; };
}

# Pipe protocol connects two routing tables. Beware of loops.
# protocol pipe {
#	table master4;		# No ipv4/ipv6 channel definition like in other protocols
#	peer table mrib4;
#	import all;		# Direction peer table -> table
#	export all;		# Direction table -> peer table
# }

# RIP example, both RIP and RIPng are supported
# protocol rip {
#	ipv4 {
#		# Export direct, static routes and ones from RIP itself
#		import all;
#		export where source ~ [ RTS_DEVICE, RTS_STATIC, RTS_RIP ];
#	};
#	interface "eth*" {
#	  	update time 10;			# Default period is 30
#		timeout time 60;		# Default timeout is 180
#		authentication cryptographic;	# No authentication by default
#		password "hello" { algorithm hmac sha256; }; # Default is MD5
#	};
# }

# OSPF example, both OSPFv2 and OSPFv3 are supported
# protocol ospf v3 {
#  	ipv6 {
#		import all;
#		export where source = RTS_STATIC;
#	};
#	area 0 {
#		interface "eth*" {
#			type broadcast;		# Detected by default
#			cost 10;		# Interface metric
#			hello 5;		# Default hello perid 10 is too long
#		};
#		interface "tun*" {
#			type ptp;		# PtP mode, avoids DR selection
#			cost 100;		# Interface metric
#			hello 5;		# Default hello perid 10 is too long
#		};
#		interface "dummy0" {
#			stub;			# Stub interface, just propagate it
#		};
#	};
#}

# Define simple filter as an example for BGP import filter
# See https://gitlab.labs.nic.cz/labs/bird/wikis/BGP_filtering for more examples
# filter rt_import
# {
#	if bgp_path.first != 64496 then accept;
#	if bgp_path.len > 64 then accept;
#	if bgp_next_hop != from then accept;
#	reject;
# }

# BGP example, explicit name 'uplink1' is used instead of default 'bgp1'
# protocol bgp uplink10 {
#	description "My BGP uplink";
#	local 192.168.10.205 as 64516;
#	neighbor 192.168.10.11 as 64514; #64515;
#	#neighbor 192.168.10.12 as 64514;
#      ipv4 {
#              import all;
#              export all;
#      };

#	hold time 90;		# Default is 240
#	password "secret";	# Password used for MD5 authentication
#
#	ipv4 {			# regular IPv4 unicast (1/1)
#		import filter rt_import;
#		export where source ~ [ RTS_STATIC, RTS_BGP ];
#	};
#
#	ipv6 {			# regular IPv6 unicast (2/1)
#		import filter rt_import;
#		export filter {	# The same as 'where' expression above
#			if source ~ [ RTS_STATIC, RTS_BGP ]
#			then accept;
#			else reject;
#		};
#	};
#
#	ipv4 multicast {	# IPv4 multicast topology (1/2)
#		table mrib4;	# explicit IPv4 table
#		import filter rt_import;
#		export all;
#	};
#
#	ipv6 multicast {	# IPv6 multicast topology (2/2)
#		table mrib6;	# explicit IPv6 table
#		import filter rt_import;
#		export all;
#	};
#}

 protocol bgp uplink2 {
        description "My BGP uplink";
        local 192.168.10.205 as 64516;
#        neighbor 192.168.10.11 as 64514; #64515;
      neighbor 192.168.10.12 as 64514;
      ipv4 {
              import all;
              export all;
      };

}

# Template example. Using templates to define IBGP route reflector clients.
# template bgp rr_clients {
#	local 10.0.0.1 as 65000;
#	neighbor as 65000;
#	rr client;
#	rr cluster id 1.0.0.1;
#
#	ipv4 {
#		import all;
#		export where source = RTS_BGP;
#	};
#
#	ipv6 {
#		import all;
#		export where source = RTS_BGP;
#	};
# }
#
# protocol bgp client1 from rr_clients {
#	neighbor 10.0.1.1;
# }
#
# protocol bgp client2 from rr_clients {
#	neighbor 10.0.2.1;
# }
#
# protocol bgp client3 from rr_clients {
#	neighbor 10.0.3.1;
# }
```

konfigurasinya cukup simpel yaitu fokus pada bagian uplink BGP

```
 protocol bgp uplink2 {
        description "My BGP uplink";
        local 192.168.10.205 as 64516;
      neighbor 192.168.10.12 as 64514;
      ipv4 {
              import all;
              export all;
      };

}
```
Note: dimana neighbornya nanti diarahkan ke network kubernetes


### Install Kubernetes 

untuk instalasi kubernetes bisa ikuti [Tutorial ini](https://alanadiprastyo.github.io/2021/10/28/install-and-configure-ha-kubernetes-cluster-using-keepalived-and-haproxy.html)

### Konfigurasi BGP Peer

- Download Calicoctl 
```
curl -o calicoctl -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.21.0/calicoctl"
```

kemudian tambahkan ASN `64514` pada semua node
```
./calicoctl patch node worker01-k8s -p '{"spec": {"bgp": {"asNumber": "64514"}}}'
./calicoctl patch node master-k8s -p '{"spec": {"bgp": {"asNumber": "64514"}}}'
```

Jalankan manifest BGPPeer dari k8s ke router bird
```
./calicoctl apply -f - << EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64516
spec:
  peerIP: 192.168.10.205
  asNumber: 64516
EOF
```

cek status BGP pada calico pastikan established
```
./calicoctl node status
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+----------+--------------------------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |              INFO              |
+----------------+-------------------+-------+----------+--------------------------------+
| 192.168.10.205 | global            | up    | 15:48:09 | Established                    |
| 192.168.10.12  | node-to-node mesh | up    | 15:45:59 | Established                    |
+----------------+-------------------+-------+----------+--------------------------------+
```
kemudian pasitkan pada Router BGP Bird sudah ada update networknya kubernetes

```
[root@bird-secondary ~]# ip route
default via 192.168.122.1 dev eth0 proto static metric 100 
10.244.108.0/26 via 192.168.10.12 dev eth1 proto bird metric 32 
10.244.150.192/26 via 192.168.10.12 dev eth1 proto bird metric 32 
192.168.10.0/24 dev eth1 proto kernel scope link src 192.168.10.205 metric 103 
192.168.122.0/24 dev eth0 proto kernel scope link src 192.168.122.205 metric 100 

[root@bird-secondary ~]# birdc show route
BIRD 2.0.8 ready.
Table master4:
10.244.150.192/26    unicast [uplink2 2021-11-16] * (100) [AS64514i]
	via 192.168.10.12 on eth1
10.244.108.0/26      unicast [uplink2 2021-11-16] * (100) [AS64514i]
	via 192.168.10.12 on eth1
```

kemudian tambahkan BGP konfigurasi unutk network service jika diperlukan
```
./calicoctl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
EOF
```
dan pastikan lagi network yg sudah ditambahkan diatas sudah advertise di sisi Router BGP

```
[root@bird-secondary ~]# birdc show route
BIRD 2.0.8 ready.
Table master4:
10.49.0.0/16         unicast [uplink2 11:18:40.570] * (100) [AS64514i]
	via 192.168.10.12 on eth1
10.244.150.192/26    unicast [uplink2 11:18:40.570] * (100) [AS64514i]
	via 192.168.10.12 on eth1
10.244.108.0/26      unicast [uplink2 11:18:40.570] * (100) [AS64514i]
	via 192.168.10.12 on eth1
[root@bird-secondary ~]# ip route
default via 192.168.122.1 dev eth0 proto static metric 100 
10.49.0.0/16 via 192.168.10.12 dev eth1 proto bird metric 32 
10.244.108.0/26 via 192.168.10.12 dev eth1 proto bird metric 32 
10.244.150.192/26 via 192.168.10.12 dev eth1 proto bird metric 32 
192.168.10.0/24 dev eth1 proto kernel scope link src 192.168.10.205 metric 103 
192.168.122.0/24 dev eth0 proto kernel scope link src 192.168.122.205 metric 100 
```
kemudian deploy nginx untuk memastikan network pod bisa langsung diakses dari network BGP
```
 kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
verifikasi ip pada pod
```
[root@master01-k8s ~]# kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
nginx-7848d4b86f-r2p54   1/1     Running   0          45h   10.244.150.195   worker01-k8s   <none>           <none>
```
pada pod mengunakan IP `10.244.150.195`, kemudian akses dari network BGP
```
[root@bird-secondary ~]# ping 10.244.150.195
PING 10.244.150.195 (10.244.150.195) 56(84) bytes of data.
64 bytes from 10.244.150.195: icmp_seq=1 ttl=63 time=0.462 ms
^C
--- 10.244.150.195 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.462/0.462/0.462/0.000 ms
[root@bird-secondary ~]# curl 10.244.150.195
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```




Ref:
- https://www.tigera.io/blog/advertising-kubernetes-service-ips-with-calico-and-bgp/
- https://docs.projectcalico.org/networking/bgp
- https://docs.projectcalico.org/reference/architecture/design/l2-interconnect-fabric
- https://octetz.com/docs/2018/2018-12-10-route-reflectors-in-calico/
- https://www.bbva.com/en/seamless-networking-for-container-orchestration-engines/
- https://createnetech.tistory.com/53
- https://bsdrp.net/documentation/examples/bgp_route_reflector_and_confederation_using_quagga_and_bird
- https://docs.cilium.io/en/v1.8/gettingstarted/bird/

