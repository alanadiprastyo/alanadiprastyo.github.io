# Install Ceph with Cephadm on CentOS 7

## Requirements
### Node Planning

| Hostname 	| Public Network 	| Cluster Network 	| role 	| Disk 	|
|---	|---	|---	|---	|---	|
| storage-ceph-01 	| 10.0.21.16/24 	| 172.16.100.16/24 	| admin-node，mon，mgr，osd 	| OS disk: vda; data disk: vdb 	|
| storage-ceph-02 	| 10.0.21.17/24 	| 172.16.100.17/24 	| mon, mgr, osd 	| OS disk: vda; data disk: vdb 	|
| storage-ceph-03 	| 10.0.21.18/24 	| 172.16.100.18/24 	| mon, mgr, osd 	| OS disk: vda; data disk: vdb 	|

### Software Requirements
- Python 3
- Systemd
- Podman or Docker for running containers
- Time synchronization (such as chrony or NTP)
- LVM2 for provisioning storage devices

Note: Any modern Linux distribution should be sufficient. Dependencies are installed automatically by the bootstrap process below.

### Recommended methods
Cephadm installs and manages a Ceph cluster using containers and systemd, with tight integration with the CLI and dashboard GUI.

- cephadm only supports Octopus and newer releases.
- cephadm is fully integrated with the new orchestration API and fully supports the new CLI and dashboard features to manage cluster deployment.
- cephadm requires container support (podman or docker) and Python 3.

Rook deploys and manages Ceph clusters running in Kubernetes, while also enabling management of storage resources and provisioning via Kubernetes APIs. We recommend Rook as the way to run Ceph in Kubernetes or to connect an existing Ceph storage cluster to Kubernetes.

- Rook only supports Nautilus and newer releases of Ceph.
- Rook is the preferred method for running Ceph on Kubernetes, or for connecting a Kubernetes cluster to an existing (external) Ceph cluster.
- Rook supports the new orchestrator API. New management features in the CLI and dashboard are fully supported.


*Note: in this tutorial will use cephadm*

----

`cephadm` deploys and manages a Ceph cluster. It does this by connecting the manager daemon to hosts via **SSH**. The manager daemon is able to add, remove, and update Ceph containers. `cephadm` does not rely on external configuration tools such as Ansible, Rook, and Salt.

`cephadm` manages the full lifecycle of a Ceph cluster. This lifecycle starts with the bootstrapping process, when `cephadm` creates a tiny Ceph cluster on a single node. This cluster consists of one monitor and one manager. cephadm then uses the orchestration interface (“day 2” commands) to expand the cluster, adding all hosts and provisioning all Ceph daemons and services. Management of this lifecycle can be performed either via the Ceph command-line interface (CLI) or via the dashboard (GUI).

`cephadm` is new in Ceph release v15.2.0 (Octopus) and does not support older versions of Ceph.

### Install Package Dependencies

Install package on all nodes
```bash
#install python3, docker, chrony and lvm2
yum install epel-release -y
yum install python3 docker chrony lvm2 -y

#enable and start service docker 
systemctl enable docker
systemctl start docker
systemctl status docker

#check chrony
chronyc sources

#set timezone to Asia/Jakarta
timedatectl set-timezone Asia/Jakarta
```
*Note: make sure all node have same time with sync to ntp server*

### Install Cephadm
install cephadm on node storage-ceph-01

```bash
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release octopus
```
```bash
$ ./cephadm install
Installing packages ['cephadm']...
Non-zero exit code 1 from yum install -y cephadm
yum: stdout Loaded plugins: fastestmirror, langpacks, priorities
yum: stdout Loading mirror speeds from cached hostfile
yum: stdout  * base: pxe.dev.purestorage.com
yum: stdout  * centosplus: pxe.dev.purestorage.com
yum: stdout  * epel: mirror.lax.genesisadaptive.com
yum: stdout  * extras: pxe.dev.purestorage.com
yum: stdout  * updates: pxe.dev.purestorage.com
yum: stdout 279 packages excluded due to repository priority protections
yum: stdout Resolving Dependencies
yum: stdout --> Running transaction check
yum: stdout ---> Package cephadm.noarch 2:15.2.14-0.el7 will be installed
yum: stdout --> Finished Dependency Resolution
yum: stdout
yum: stdout Dependencies Resolved
yum: stdout
yum: stdout ================================================================================
yum: stdout  Package        Arch          Version                  Repository          Size
yum: stdout ================================================================================
yum: stdout Installing:
yum: stdout  cephadm        noarch        2:15.2.14-0.el7          Ceph-noarch         55 k
yum: stdout
yum: stdout Transaction Summary
yum: stdout ================================================================================
yum: stdout Install  1 Package
yum: stdout
yum: stdout Total download size: 55 k
yum: stdout Installed size: 223 k
yum: stdout Downloading packages:
yum: stdout Public key for cephadm-15.2.14-0.el7.noarch.rpm is not installed
yum: stdout Retrieving key from https://download.ceph.com/keys/release.gpg
yum: stderr warning: /var/cache/yum/x86_64/7/Ceph-noarch/packages/cephadm-15.2.14-0.el7.noarch.rpm: Header V4 RSA/SHA256 Signature, key ID    460f3994: NOKEY
yum: stderr
yum: stderr
yum: stderr Invalid GPG Key from https://download.ceph.com/keys/release.gpg: No key found in given key data
Traceback (most recent call last):
  File "./cephadm", line 8432, in <module>
    main()
  File "./cephadm", line 8420, in main
    r = ctx.func(ctx)
  File "./cephadm", line 6384, in command_install
    pkg.install(ctx.packages)
  File "./cephadm", line 6231, in install
    call_throws(self.ctx, [self.tool, 'install', '-y'] + ls)
  File "./cephadm", line 1461, in call_throws
    raise RuntimeError('Failed command: %s' % ' '.join(command))
RuntimeError: Failed command: yum install -y cephadm
```
Based on (Ceph Documentation)[https://docs.ceph.com/en/mimic/install/get-packages/] , execute the following to install the release.asc key.
```bash
$ rpm --import 'https://download.ceph.com/keys/release.asc'
```
install cephadm package again and it succeeds.
```bash
$ ./cephadm install
Installing packages ['cephadm']...
   
$ which cephadm
/usr/sbin/cephadm
```

### Install Ceph CLI
```bash
cephadm prepare-host
cephadm install ceph-common ceph-osd
```
### Create new cluster
This command apply on storage-ceph-01
```bash
cephadm bootstrap --mon-ip 10.0.21.16
```
Note: it will be error if ssh port custom, so to solve this problem we can change the ssh configuration
```bash
Host *
  StrictHostKeyChecking no
  Port                  2222
```
```bash
ceph cephadm set-ssh-config -i ssh_config
ceph cephadm get-ssh-config
---
Host *
  StrictHostKeyChecking no
  Port                  2222
---

ceph orch host add storage-ceph-01
ceph orch host ls
```
This command will:

1. Create monitor and manager daemons on the local host for the new cluster
2. Generate a new SSH key for Ceph cluster and add it to / root /. SSH / authorized of root user_ In the keys file
3. Write the minimum configuration file to / etc/ceph/ceph.conf
4. Will client.admin Management privilege key write to / etc/ceph/ceph.client.admin.keyring
5. Write the public key to / etc/ceph/ceph.pub

### Add hosts
add nodes **storage-ceph-02 (10.0.21.17)** and **storage-ceph-03 (10.0.21.18)**

in the authorized_ Install the public SSH key of the cluster in the keys file
```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.21.17
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.21.18
```
and then add node with pattern `ceph orch host add hostname ip-cluster-public`
```bash
ceph orch host add storage-ceph-02 10.0.21.17
ceph orch host add storage-ceph-03 10.0.21.18
```
Or through YAML file, you can use CEPH arch apply - I host.yml Add more than one host at a time
```bash
vi host.yaml
```
```yaml
---
service_type: host
addr: 10.0.21.17
hostname: storage-ceph-02
labels:
  - ceph02_mon
---
service_type: host
addr: 10.0.21.18
hostname: storage-ceph-03
labels:
  - ceph03_mon
```

### Check Hosts
```bash
[root@storage-ceph-01 ~]# ceph orch host ls
HOST             ADDR             LABELS  STATUS  
storage-ceph-01  storage-ceph-01                  
storage-ceph-02  10.0.21.17                       
storage-ceph-03  10.0.21.18
```

### Add Ceph Mon
A typical Ceph cluster has three or five monitoring daemons distributed on different hosts. If there are five or more nodes in the cluster, we recommend deploying five monitors.

When the cluster expands, Ceph automatically deploys the monitor daemons, while when the cluster shrinks, Ceph automatically shrinks the monitor daemons. The smooth implementation of this automatic growth and contraction depends on the correct subnet configuration.

The cephadm boot process assigns the first monitor Daemon in the cluster to a specific subnet, which cephadm specifies as the default subnet of the cluster.

Deploy monitors to specific host only
```bash
ceph orch apply mon "storage-ceph-01,storage-ceph-02,storage-ceph-03"
```
*Note: Make sure the first host (boot) is included in this list.*

Or through YAML file, you can use CEPH arch apply - I mon.yml Add more than one monitor at a time
```bash
vi mon.yaml
```
```yaml
---
service_type: mon
placement:
  hosts:
    - storage-ceph-01
    - storage-ceph-02
    - storage-ceph-03
```
### Deploy OSD
In order to deploy OSD, there must be at least one available storage device.
```bash
[root@storage-ceph-01 ~]# ceph orch device ls
Hostname         Path      Type  Serial  Size   Health   Ident  Fault  Available  
storage-ceph-01  /dev/vdb  hdd            107G  Unknown  N/A    N/A    No         
storage-ceph-02  /dev/vdb  hdd            107G  Unknown  N/A    N/A    No         
storage-ceph-03  /dev/vdb  hdd            107G  Unknown  N/A    N/A    No  
```
The storage device is considered available if all of the following conditions are met
1) The device must have no partition
2) The device must not have any LVM status
3) Equipment must not be mounted
4) The device must not contain a file system
5) The device must not contain Ceph BlueStore OSD
6) The device must be larger than 5 GB

Note: this ref => https://docs.ceph.com/en/latest/cephadm/services/osd/

Create OSD use all available and unused storage devices
```bash
ceph orch apply osd --all-available-devices
```
Note: if you want to specific devices on a specific host `ceph orch daemon add osd *<host>*:*<device-path>*` for example `ceph orch daemon add osd ceph01:/dev/sdb`

### check cluster status
```bash
[root@storage-ceph-01 ~]# ceph -s
  cluster:
    id:     f42d0246-3658-11ec-81d2-525400e44f9a
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum storage-ceph-01,storage-ceph-02,storage-ceph-03 (age 24h)
    mgr: storage-ceph-01.pnwelp(active, since 25h), standbys: storage-ceph-02.tnvlub
    osd: 3 osds: 3 up (since 25h), 3 in (since 25h)
    rgw: 1 daemon active (default.us-east-1.storage-ceph-01.qgezbn)
```

### Add module dashboard to easy manage the cluster
1. Enable dashboard module
```bash
ceph mgr module enable dashboard
```
2. generate and install a self signed certificate
```bash
ceph dashboard create-self-signed-cert
```
3. crete password
```bash
vi password.txt
mypassword

ceph dashboard ac-user-create admin -i password.txt administrator
```
4. Modify the access IP of the cluster dashboard
```bash
ceph config-key set mgr/dashboard/server_addr 0.0.0.0
```
View service access mode
```bash
ceph mgr services
{
    "dashboard": "https://storage-ceph-01:8443/"
}
```
Access to `https://10.0.21.16:8443/#/login`
![Login Ceph](/img/ceph/login-dashboard-ceph.png)
Access to Dashboard
![Dashboard Ceph](/img/ceph/ceph-dashboard.png)

Ref:
- https://docs.ceph.com
- https://www.fatalerrors.org/a/cephadm-deploying-ceph-cluster.html
- https://www.flamingbytes.com/posts/deploy-ceph-cluster/
