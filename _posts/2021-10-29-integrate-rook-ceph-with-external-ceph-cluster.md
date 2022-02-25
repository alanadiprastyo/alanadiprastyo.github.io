# Integrate Rook Ceph with External Ceph Cluster

### Setup Ceph Cluster with kubeadm
- https://docs.ceph.com/en/latest/cephadm/install/
- https://www.fatalerrors.org/a/cephadm-deploying-ceph-cluster.html
- https://alanadiprastyo.github.io/2021/10/27/install-ceph-with-cephadm.html

Create Pool -> replicated_2g

### Deploy the Rook Operator
Ref: https://rook.io/docs/rook/v1.7/ceph-cluster-crd.html#external-cluster

A simple Rook cluster can be created with the following kubectl commands and example manifests.
```bash
git clone --single-branch --branch release-1.7 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f crds.yaml -f common.yaml -f operator.yaml

# verify the rook-ceph-operator is in the `Running` state before proceeding
kubectl -n rook-ceph get pod
```

make sure pod operator rook ceph running

Get the FSID and other details needed for the helper script from the ceph cluster
```bash
ceph health
ceph fsid
ceph auth get-key client.admin
ceph status
```
output
```bash
[root@storage-ceph-01 ~]# ceph health
HEALTH_OK
[root@storage-ceph-01 ~]# ceph fsid

f42d0246-3658-11ec-81d2-525400e44f9a
[root@storage-ceph-01 ~]# ceph auth get-key client.admin
AQDP9Xdh4vHmERAAZPBaDyffrct5eDkPe6mbjg==
[root@storage-ceph-01 ~]# ceph status
  cluster:
    id:     f42d0246-3658-11ec-81d2-525400e44f9a
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum storage-ceph-01,storage-ceph-02,storage-ceph-03 (age 14h)
    mgr: storage-ceph-01.pnwelp(active, since 14h), standbys: storage-ceph-02.tnvlub
    osd: 3 osds: 3 up (since 14h), 3 in (since 14h)
    rgw: 1 daemon active (default.us-east-1.storage-ceph-01.qgezbn)
 
  task status:
 
  data:
    pools:   6 pools, 137 pgs
    objects: 228 objects, 5.4 KiB
    usage:   3.2 GiB used, 297 GiB / 300 GiB avail
    pgs:     137 active+clean
```

running common-external to create rolebinding, serviceaccount innamespaces rook-ceph-external

```bash
kubectl create -f common-external.yaml
```

export variable to running file import-external-cluster.sh
```bash
export NAMESPACE=rook-ceph-external
export ROOK_EXTERNAL_FSID=f42d0246-3658-11ec-81d2-525400e44f9a
export ROOK_EXTERNAL_CEPH_MON_DATA=a=10.0.21.16:3300,b=10.0.21.17:3300,c=10.0.21.18:3300
export ROOK_EXTERNAL_ADMIN_SECRET=AQDP9Xdh4vHmERAAZPBaDyffrct5eDkPe6mbjg==

bash import-external-cluster.sh
```

Edit secret rook-ceph-mon to delete ceph-secret and ceph-username

```bash
kubectl edit secret rook-ceph-mon -n rook-ceph-external
```
output
```yaml
apiVersion: v1
data:
  admin-secret: QVFEUDlYZGg0dkhtRVJBQVpQQmFEeWZmcmN0NWVEa1BlNm1iamc9PQ==
  cluster-name: cm9vay1jZXBoLWV4dGVybmFs
  fsid: ZjQyZDAyNDYtMzY1OC0xMWVjLTgxZDItNTI1NDAwZTQ0Zjlh
  mon-secret: bW9uLXNlY3JldA==
kind: Secret
metadata:
  creationTimestamp: "2021-10-27T03:40:06Z"
  name: rook-ceph-mon
  namespace: rook-ceph-external
  resourceVersion: "4234"
  uid: 536f3405-3255-4065-ae4d-08e719d0737b
type: kubernetes.io/rook
```

apply manifest cluster-external to create initial CephCluster
```bash
kubectl apply -f cluster-external.yaml
kubectl get CephCluster -n rook-ceph-external
```
output
```bash
NAME                 DATADIRHOSTPATH   MONCOUNT   AGE    PHASE       MESSAGE                          HEALTH      EXTERNAL
rook-ceph-external                                2m4s   Connected   Cluster connected successfully   HEALTH_OK   true
```

make sure status connected

- Create Storage Class for Ceph RBD

we will use pool replicated_2g, and create manifest for storageclass ceph external

```yaml
# vi sc-ceph.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-ext
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
   # clusterID is the namespace where the rook cluster is running
   clusterID: rook-ceph-external
   # Ceph pool into which the RBD image shall be created
   pool: replicated_2g

   # RBD image format. Defaults to "2".
   imageFormat: "2"

   # RBD image features. Available for imageFormat: "2". CSI RBD currently supports only `layering` feature.
   imageFeatures: layering

   # The secrets contain Ceph admin credentials.
   csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
   csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph-external
   csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
   csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph-external
   csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
   csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph-external

   # Specify the filesystem type of the volume. If not specified, csi-provisioner
   # will set default as `ext4`. Note that `xfs` is not recommended due to potential deadlock
   # in hyperconverged settings where the volume is mounted on the same node as the osds.
   csi.storage.k8s.io/fstype: ext4

# Delete the rbd volume when a PVC is deleted
reclaimPolicy: Delete
allowVolumeExpansion: true
```
create storageclass manifest

```bash
kubectl apply -f sc-ceph.yaml
kubectl get sc
```
output
```bash
NAME                  PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block-ext   rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   36s
```

create pvc
```yaml
kubectl create ns test
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: ceph-ext
 namespace: test
 labels:
   app: nginx
spec:
 storageClassName: rook-ceph-block-ext
 accessModes:
 - ReadWriteOnce
 resources:
   requests:
    storage: 1Gi 
EOF
```
check pv and pvc
```bash
# kubectl get pvc,pv -n test
NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/ceph-ext   Bound    pvc-534aa0e5-84b6-466e-84fd-065deed4b2d9   1Gi        RWO            rook-ceph-block-ext   31s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           STORAGECLASS          REASON   AGE
persistentvolume/pvc-534aa0e5-84b6-466e-84fd-065deed4b2d9   1Gi        RWO            Delete           Bound    test/ceph-ext   rook-ceph-block-ext            29s
```

in ceph dashboard we can check block rbd

create pod nginx with mount to pvc
```yaml
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  namespace: test
spec:
  volumes:
    - name: mystorage
      persistentVolumeClaim:
        claimName: ceph-ext
  containers:
    - name: task-pv-container
      #image: nginx:latest
      image: quay.io/bitnami/nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: mystorage
EOF
```
verify the pod
```bash
# kubectl get pod -n test
NAME         READY   STATUS    RESTARTS   AGE
nginx-test   1/1     Running   0          41s
```

exec to pod nginx-test to check rbd storage

```bash
# kubectl exec -it nginx-test /bin/bash -n test

1001@nginx-test:/app$ ls
50x.html  index.html
1001@nginx-test:/app$ df -h
Filesystem               Size  Used Avail Use% Mounted on
overlay                   27G  5.2G   22G  20% /
tmpfs                     64M     0   64M   0% /dev
tmpfs                    7.8G     0  7.8G   0% /sys/fs/cgroup
shm                       64M     0   64M   0% /dev/shm
tmpfs                    7.8G   13M  7.8G   1% /etc/passwd
/dev/mapper/centos-root   27G  5.2G   22G  20% /etc/hosts
/dev/rbd0                976M  2.6M  958M   1% /usr/share/nginx/html
tmpfs                     16G   12K   16G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                    7.8G     0  7.8G   0% /proc/acpi
tmpfs                    7.8G     0  7.8G   0% /proc/scsi
tmpfs                    7.8G     0  7.8G   0% /sys/firmware
```

- Create Storage Class for Ceph Filesystem

Before apply configuration storage class on kubernetes, we must create Filesystem on external Ceph Cluster

1. Create Filesystem on ceph cluster
```bash
ceph fs volume create alan-mds --placement="3"
```

verify on ceph dashboard
![Topologi](/img/ceph/fs.png)

and also verify pool on ceph dashboard
![Topologi](/img/ceph/pool-fs.png)

Create manifest for storage class with name sc-ceph-fs.yaml
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  # clusterID is the namespace where the rook cluster is running
  # If you change this namespace, also change the namespace below where the secret namespaces are defined
  clusterID: rook-ceph-external

  # CephFS filesystem name into which the volume shall be created
  fsName: alan-mds
  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: cephfs.alan-mds.data
  #pool: awanio-cephfs 

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  #csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph-external
  #csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph-external
  #csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph-external

reclaimPolicy: Delete
allowVolumeExpansion: true
```

on manifest we must make sure some configuration like <b>clusterID</b>, <b>fsName</b> and <b>pool</b> 

Apply the manifest
```bash
kubectl apply -f sc-ceph-fs.yaml
```

verify storage cluster
```bash
# kubectl get sc
NAME                            PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn                        driver.longhorn.io              Delete          Immediate           true                   51d
rook-ceph-block-ext (default)   rook-ceph.rbd.csi.ceph.com      Delete          Immediate           true                   45d
rook-cephfs                     rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   22h
```

Example create pvc use storage class rook-cephfs use kube-registry pod with the shared filesystem as the backing store

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
  namespace: kube-system
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-registry
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: kube-registry
  template:
    metadata:
      labels:
        k8s-app: kube-registry
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: registry
        image: registry:2
        imagePullPolicy: Always
        resources:
          limits:
            cpu: 100m
            memory: 100Mi
        env:
        # Configuration reference: https://docs.docker.com/registry/configuration/
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_HTTP_SECRET
          value: "Ple4seCh4ngeThisN0tAVerySecretV4lue"
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
        volumeMounts:
        - name: image-store
          mountPath: /var/lib/registry
        ports:
        - containerPort: 5000
          name: registry
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: registry
        readinessProbe:
          httpGet:
            path: /
            port: registry
      volumes:
      - name: image-store
        persistentVolumeClaim:
          claimName: cephfs-pvc
          readOnly: false
```
```bash
kubectl apply -f kube-registry-cephfs.yaml
```
verify deployment kube-registry
```bash
# kubectl get pvc -n kube-system
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cephfs-pvc   Bound    pvc-b365e347-fdfc-4d91-8209-a9d9f003b658   1Gi        RWX            rook-cephfs    34s

# kubectl exec -it kube-registry-5b677b6c87-qnwjt /bin/sh -n kube-system
# df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                 776.7G    311.2G    433.4G  42% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                   125.8G         0    125.8G   0% /sys/fs/cgroup
shm                      64.0M         0     64.0M   0% /dev/shm
/dev/mapper/centos_awid9-root
                        776.7G    311.2G    433.4G  42% /etc/hosts
/dev/mapper/centos_awid9-root
                        776.7G    311.2G    433.4G  42% /dev/termination-log
/dev/mapper/centos_awid9-root
                        776.7G    311.2G    433.4G  42% /etc/hostname
/dev/mapper/centos_awid9-root
                        776.7G    311.2G    433.4G  42% /etc/resolv.conf
10.22.129.15:6789,10.22.129.16:6789,10.22.129.20:6789:/volumes/csi/csi-vol-88802b54-964b-11ec-96b5-3625ccda6e97/80d3f96b-809f-4623-b21b-4024a7c81c8a
                          1.0G         0      1.0G   0% /var/lib/registry
tmpfs                   100.0M     12.0K    100.0M   0% /run/secrets/kubernetes.io/serviceaccount
tmpfs                   125.8G         0    125.8G   0% /proc/acpi
tmpfs                    64.0M         0     64.0M   0% /proc/kcore
tmpfs                    64.0M         0     64.0M   0% /proc/keys
tmpfs                    64.0M         0     64.0M   0% /proc/timer_list
tmpfs                    64.0M         0     64.0M   0% /proc/timer_stats
tmpfs                    64.0M         0     64.0M   0% /proc/sched_debug
tmpfs                   125.8G         0    125.8G   0% /proc/scsi
tmpfs                   125.8G         0    125.8G   0% /sys/firmware
```

Ref:
- https://rook.io/docs/rook/v1.8/ceph-filesystem.html