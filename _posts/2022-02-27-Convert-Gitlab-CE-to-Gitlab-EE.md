# Convert Community Edition to Enterprise Edition

Untuk mengkonversi Gitlab Community Edition (CE) yang sudah terinstall pada server mengunakan package Omnibus Gitlab ke Gitlab Enterprise Edition (EE), kita perlu install package EE diatas versi CE

Asumsi kita melakukan upgrade mengunakan versi yang sama, sebagai contoh (CE 12.1 ke EE 12.1) ini meruapakan mekanisme upgrade yang <b>sangat direkomendasikan</b>

sebelum memulai tutorial ini anda bisa mengikuti tutorial ini untuk install Gitlab CE [How to Install Centos CE on CentOS 7](https://alanadiprastyo.github.io/2022/02/27/How-to-install-Gitlab-CE-on-Centos-7.html)

- Untuk memulai konversi versi CE ke EE kita perlu memverifikasi versi Gitlab yang kita gunakan

```bash
$ sudo rpm -q gitlab-ce
gitlab-ce-14.8.2-ce.0.el7.x86_64
```
- Menambahkan repositori untuk versi EE
```bash
curl --silent "https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh" | sudo bash
```

Berikut perintah untuk melihat semua versi yang ada pada repositori
```bash
yum --showduplicates list gitlab-ee
```

- Selanjutnya, instal package gitlab-ee. pada perintah ini nantinya package gitlab-ce akan terhapus otomatis  pada gitlab server, dan  pastikan setelah proses intalasi gitlab ee berhasil, jalankan perintah `gitlab-ctl reconfigure`. 

<b>Pastikan kita menginstall versi Gitlab yang sama dengan versi sebelunya pada versi CE</b>

```bash
sudo yum install gitlab-ee-14.8.2-ee.0.el7.x86_64
```
![Replace Gitlab](/img/gitlab/replace-gitlab.png)

![Replace Gitlab](/img/gitlab/succes-upgrade.png)

Note: pada saat instalasi Gitlab EE, disitu akan diinformasikan kita akan replace versi CE

- Jalankan reconfigure pada Gitlab EE
```bash
sudo gitlab-ctl reconfigure
```
- kemudian akses dashboard

Admin > Setting > General > License file
![Replace Gitlab](/img/gitlab/upload-lisensi.png)
Jika proses verifikasi lisensi berhasil maka akan di informasikan jenis subscription yang kita gunakan

![Replace Gitlab](/img/gitlab/result-subs.png)

- Terkahir setelah proses upgrade berjalan dengan baik, maka berikutnya hapus repositori gitlab CE
```bash
sudo rm /etc/yum.repos.d/gitlab_gitlab-ce.repo
```

Ref:
- https://docs.gitlab.com/ee/update/package/convert_to_ee.html