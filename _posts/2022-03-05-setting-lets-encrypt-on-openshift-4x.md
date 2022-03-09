https://stephennimmo.com/securing-openshift-ingress-with-lets-encrypt/
# Setting Let's Encrypt on Openshift 4.x

Untuk memulai mengunakan sertifikat SSL Letsencrypt perlu mengunakan domain yang diregister secara publik, untuk tutorial kali ini saya mengunakan domain <b>cloudnusantara.my.id</b> yang dimanage didalam google cloud

### Penambahan A Record
![Login Gitlab](/img/ocp/dns-ocp.png)

Note: biasanya ketika kita install ocp diatas cloud provider seperti gcp, record ini akan terbuat secara otomatis

### Instalasi Certbot dan Request Certificate wildcard *.apps

Instalasi certbot akan dijalankan pada bastion seperti dibawah ini
```bash
sudo yum install epel-release
sudo yum install certbot
```
jalankan certbot untuk meminta certificate wildcard *.apps.okd.cloudnusantara.my.id
```
sudo  certbot -d '*.apps.okd.cloudnusantara.my.id' --manual --preferred-challenges dns certonly
```
ketika menjalankan ini maka akan muncul perintah untuk membuat domain TXT
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Requesting a certificate for *.apps.okd.cloudnusantara.my.id
Performing the following challenges:
dns-01 challenge for apps.okd.cloudnusantara.my.id
 
Please deploy a DNS TXT record under the name
_acme-challenge.apps.okd.cloudnusantara.my.id with the following value:
******-******-******
Before continuing, verify the record is deployed.
 
Press Enter to Continue
```
Note: Jakan tekan enter pada bagian ini, tapi buat record TXT seperti dibawah ini.

![Login Gitlab](/img/ocp/dns-txt.png)

sebelum tekan enter, maka test terlebih dahulu apakah record txt sudah terdaftar atau belum, dengan menjalankan perintah dibawah ini

```bash
dig -t txt +short _acme-challenge.apps.okd.cloudnusantara.my.id                                                alan@alan-ThinkPad-L380
```

Jika sudah sesuai, maka selanjutnya tekan enter maka jika berjalan dengan baik akan muncul result seperti dibawah ini

```bash
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/apps.okd.cloudnusantara.my.id/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/apps.okd.cloudnusantara.my.id/privkey.pem
   Your certificate will expire on 2022-06-03. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again. To non-interactively renew *all* of your
   certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

## Update Certificate pada Openshift IngressController

untuk menjalankan perintah bisa mengunakan refrensi dari dokumentasi openshift 
- https://docs.openshift.com/container-platform/4.8/security/certificates/replacing-default-ingress-certificate.html

pada bastion, jalankan perintah dibawah ini untuk membuat configmap, secret dan apply sertifikat baru kedalam ocp ingress controller

```bash
oc create configmap letsencrypt-ca-05032022 --from-file=ca-bundle.crt=/etc/letsencrypt/live/apps.okd.cloudnusantara.my.id/fullchain.pem -n openshift-config
oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"letsencrypt-ca-05032022"}}}'
oc create secret tls letsencrypt-ca-secret-05032022 --cert=/etc/letsencrypt/live/apps.okd.cloudnusantara.my.id/fullchain.pem  --key=/etc/letsencrypt/live/apps.okd.cloudnusantara.my.id/privkey.pem -n openshift-ingress
oc patch ingresscontroller.operator default --type=merge -p '{"spec":{"defaultCertificate": {"name": "letsencrypt-ca-secret-05032022"}}}' -n openshift-ingress-operator
```
Setelah menjalankan ini membutuhkan waktu beberapa menit supaya semua konfigurasi terpasang pada cluster ocp

## Verifikasi SSL
untuk memverifikasi SSL sudah terpasang bisa langsung akses dashboard OCP atau membuat route aplikasi mengunakan SSL

![Login Gitlab](/img/ocp/verify-ssl.png)

Ref: https://stephennimmo.com/securing-openshift-ingress-with-lets-encrypt/