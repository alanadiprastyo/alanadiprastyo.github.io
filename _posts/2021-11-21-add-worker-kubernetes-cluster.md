# Add Worker Kubernetes Cluster

---

Catatan: Tulisan ini hanya untuk pribadi jadi mohon maaf jika kurang rapih dan sistematis

---

Pada saat kita initial token pada saat membuat cluster kubernetes akan terbentuk sebuah token dengan masa berlaku 24 jam, maka jika kita menambahkan worker lebih dari 24 jam kita harus membuat token terlebih dahulu

## Membuat token baru

verifikasi token dengan perintah `kubeadm`
```
kubeadm token list
``` 

membuat token baru 
```
kubeadm token create --print-join-command
```
output
```
kubeadm join control-plane.k8s.example.com:6443 --token ckmnmm.3f4d1t9r5 --discovery-token-ca-cert-hash sha256:5b071e9c1a3012851bc712467a86b3e392c53d7e6ba949d30 
```

verifikasi token
```
# kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
ckmnmm.3f4d1t3a10nvi9r5   23h         2021-11-22T07:16:33Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
jr7xaq.dy2wz9njsb68qh5d   16h         2021-11-22T00:07:33Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
```

## Tambahkan Worker dengan token yang sudah dibuat

ssh kedalam worker node yang ingin ditambah kemudian jalankan perintah dibawah ini
```
kubeadm join control-plane.k8s.example.com:6443 --token ckmnmm.3f4d1t9r5 --discovery-token-ca-cert-hash sha256:5b071e9c1a3012851bc712467a86b3e392c53d7e6ba949d30
```

pastikan ketika menjalankan tidak ada error dan cek status node pada cluster
```
kubectl get nodes
```

Ref:
- https://www.serverlab.ca/tutorials/containers/kubernetes/how-to-add-workers-to-kubernetes-clusters/