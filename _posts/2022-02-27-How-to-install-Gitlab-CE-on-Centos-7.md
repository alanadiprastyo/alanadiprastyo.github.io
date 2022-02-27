# How to Install Gitlab CE on CentOS 7

Pada tutorial ini menginstall Gitlab CE dengan metode Omnibus atau install dengan single package linux dimana pada Omnibus ini terdapat kumpulan semua service yang dibutuhkan untuk membangun ekosistem service Gitlab sehingga ini memudahkan proses instalasi.

## Minimal Requirements
- CPU : minimal 4 core untuk bisa menampung user maksimal 500 user
- Memory : minimal 4 GB RAM untuk bisa menampung user maksimal 500 user
- Storage : Omnibus Gitlab minimal membutuhkan 2.5 GB, diluar dari kebutuhan untuk menyimpan repositori

## Instalasi Gitlab CE
1. Instal dan Konfigurasi service yang dibutuhkan untuk meninstall Gitlab 

Pada CentOS 7, perintah dibawah ini seperti layanan http, https dan ssh harus dibuka pada sistem firewall untuk memudahkan dalam mengakses web konsol dan git repositori

```bash
sudo yum install -y curl policycoreutils-python openssh-server perl
# Enable OpenSSH server daemon if not enabled: sudo systemctl status sshd
sudo systemctl enable sshd
sudo systemctl start sshd

# Check if opening the firewall is needed with: sudo systemctl status firewalld
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld
```

kemudian menginstal <b>postfix</b> untuk keperluan notifikasi email. Namun jika kita ingin mengunakan solusi lain untuk mengirim email ekternal SMTP seperti gmail, office 365 dan lain-lain, kita bisa melawati proses ini.

```bash
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
```
2. Menambahkan dan Install Package Repositori Gitlab
```
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```
Sebelum menginstal Gitlab, pastikan kita sudah menambahkan konfigurasi DNS Publik pada DNS Management kita. Penjelasan secara singkat cara konfigurasi DNS

### <b>Seting DNS</b>
DNS atau Domain Name System merupakan mekanisme untuk memberikan nama pada sebuah server yang memiliki IP, bertujuan untuk memudahkan kita dalam mengingat sebuah nama server.

Walaupun kita dapat menjalankan instance Gitlab dengan mengunakan IP address, namun ada beberapa kelebihan jika kita mengunakan DNS diantaranya adalah:
- Memudahkan dalam mengingat dan mengunakan Gitlab Server
- Memerlukan HTTPS

Note: <i>Keuntungan mengunakan DNS ini adalah otomatis terintegrasi dengan Let's Encrypt dengan catatan domain instancenya dapat diakses dari internet</i>

#### <b>Menambahkan A Record pada Registar Domain</b>
1. Akses dashboard management DNS kemudian tambahkan record DNS dengan tipe A Record
2. Tes bahwa konfigurasi sudah berjalan dengan baik, dengan menjalankan perintah `dig` atau `nslookup`
```bash
$ nslookup gitlab.cloudnusantara.my.id

Server:		127.0.1.1
Address:	127.0.1.1#53

Non-authoritative answer:
Name:	gitlab.cloudnusantara.my.id
Address: 35.219.66.182
```

Note: <i>"Pada kasus diatas merupakan mekanisme untuk menginstall Gitlab mengunakan DNS Publik yang secara langsung memiliki IP Publik pada Server"</i>

Kemudian jalankan perintah dibawah ini untuk instal Gitlab CE
```bash
sudo EXTERNAL_URL="https://gitlab.cloudnusantara.my.id" yum install -y gitlab-ce
```
setelah instalasi berhasil maka tahap selanjutnya adalah mengakses dashboard gitlab dengan user: root dan password akan tergenerate otomatis pada `/etc/gitlab/initial_root_password` yang akan tersimpan selama 24 jam

![Login Gitlab](/img/gitlab/login.png)

Pada kasus tutorial ini saya lupa membuka firewall pada saat instalasi sehingga tidak otomatis mengunakan sertiikat ssl let's encrypt

![Login Gitlab](/img/gitlab/cert-warning.png)

Untuk mengatasi hal tersebut kita bisa menambahkan konfigurasi pada `/etc/gitlab/gitlab.rb`

```bash
sudo vi /etc/gitlab/gitlab.rb
---
################################################################################
# Let's Encrypt integration
################################################################################
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ["alan.a.prastyo@gmail.com"] # This should be an array of email addresses to add as contacts
letsencrypt['group'] = 'root'
letsencrypt['key_size'] = 2048
letsencrypt['owner'] = 'root'
letsencrypt['wwwroot'] = '/var/opt/gitlab/nginx/www'
# See http://docs.gitlab.com/omnibus/settings/ssl.html#automatic-renewal for more on these sesttings
letsencrypt['auto_renew'] = true
letsencrypt['auto_renew_hour'] = 0
#letsencrypt['auto_renew_minute'] = nil # Should be a number or cron expression, if specified.
letsencrypt['auto_renew_day_of_month'] = "*/4"
letsencrypt['auto_renew_log_directory'] = '/var/log/gitlab/lets-encrypt'
```
kemudian jalankan perintah `gitlab-ctl reconfigure` untuk melakukan konfigurasi ulang terhadap lets encrypt
```sudo 
sudo gitlab-ctl reconfigure
```
pastikan semua service berjalan dengan baik setelah konfigurasi ulang pada sistem Gitlab
```bash
$ sudo gitlab-ctl status
run: alertmanager: (pid 7204) 28545s; run: log: (pid 6937) 28647s
run: crond: (pid 7941) 12s; run: log: (pid 6950) 243s
run: gitaly: (pid 3505) 30511s; run: log: (pid 2552) 30706s
run: gitlab-exporter: (pid 7178) 28549s; run: log: (pid 6829) 28665s
run: gitlab-kas: (pid 3465) 30517s; run: log: (pid 2880) 30690s
run: gitlab-workhorse: (pid 3475) 30517s; run: log: (pid 3087) 30605s
run: grafana: (pid 7218) 28544s; run: log: (pid 7093) 28597s
run: logrotate: (pid 3642) 1921s; run: log: (pid 2456) 30718s
run: nginx: (pid 7084) 180s; run: log: (pid 3157) 30600s
run: node-exporter: (pid 7173) 28550s; run: log: (pid 6792) 28671s
run: postgres-exporter: (pid 7212) 28544s; run: log: (pid 6977) 28641s
run: postgresql: (pid 2718) 30697s; run: log: (pid 2744) 30694s
run: prometheus: (pid 7186) 28549s; run: log: (pid 6897) 28654s
run: puma: (pid 7042) 219s; run: log: (pid 2991) 30622s
run: redis: (pid 2489) 30716s; run: log: (pid 2502) 30713s
run: redis-exporter: (pid 7180) 28550s; run: log: (pid 6861) 28660s
run: registry: (pid 6981) 238s; run: log: (pid 6980) 238s
run: sidekiq: (pid 7023) 227s; run: log: (pid 3017) 30618s
```
Akses dashboard, dan pastikan pada bagian https sudah secure mengunakan lets encrypt
![Login Gitlab](/img/gitlab/cert-secure.png)

Note: <i>Jika pada bagian https masih warning coba akses gitlab dashboard dengan web browser lain atau mengunakan incognito</i>

#### <b>Mengubah Password Default Root</b>
Karena password default bersifat generate random maka untuk memudahkan dalam pengelolaan user admin, perlu dilakukan mengubah password root

User Setting > Password > Change Password
![Login Gitlab](/img/gitlab/change-pass.png)



Ref:
- https://about.gitlab.com/install/#centos-7
- https://docs.gitlab.com/omnibus/settings/dns.html