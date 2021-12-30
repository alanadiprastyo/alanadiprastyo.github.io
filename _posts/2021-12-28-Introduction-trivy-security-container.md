# Introduction Trivy - Security Container

Trivy (tri merupakan trigger, vy merupakan envy) adalah sebuah tools yang sederhana dan secara menyeluruh untuk melakukan scanner vulnerabillity/misconfiguration untuk container dan artifact lainnya. Trivy mendeteksi vulnerabillity dari package OS (Alpine, RHEL, CentOS dan lain-lain) dan package spesifik bahasa pemrogramana (Bundler, Composer, npm, yarn dan lain-lain). sebagai tambahan trivy juga dapat melakukan scan pada Insfrastructure As Code (IAC) seperti terraform dan kubernetes untuk mendeteksi adanya masalah pada konfigurasi deployment dimana itu dapat memberikan kerentanan sebuah deployment dari serangan.

![Topologi](/img/trivy/overview.png)

## Overview
Trivy mendeteksi dua tipe dari security issue:
- Vulnerabilities
- Misconfiguration

Trivy dapat melakukan scan terhadap tiga jenis artifact:
- Contianer Images
- Filesystem dan Rootfs
- Git Repository

Trivy juga dapat berjalan pada 2 jenis mode yang berbeda:
- Standalone
- Client/Server

## Fitur
- Mendeteksi Vulnerability secara menyeluruh
  - [OS Package](https://aquasecurity.github.io/trivy/v0.22.0/vulnerability/detection/os/) (Alpine, Red Hat Universal Base Image, Red Hat Enterprise Linux, CentOS, Oracle Linux, Debian, Ubuntu, Amazon Linux, openSUSE Leap, SUSE Enterprise Linux, Photon OS dan Distroless)

    Note: Trivy tidak mendukung paket/binari yang dikompilasi sendiri, tetapi paket resmi yang disediakan oleh vendor seperti Red Hat dan Debian.

  - [Language-spesific packages](https://aquasecurity.github.io/trivy/v0.22.0/vulnerability/detection/language/) (Bundler, Composer, Pipenv, Poetry, npm, yarn, Cargo, NuGet, Maven, and Go)

- Mendeteksi Misconfigurations IAC
  - Sudah termasuk [built-in-policies](https://aquasecurity.github.io/trivy/v0.22.0/misconfiguration/policy/builtin/)
    - Kubernetes
    - Docker
    - Terraform
    - more coming soon
- Sederhana
- Cepat
  - Cepat dalam scan dimana akan selesai dalam waktu 10 detik (tergantung dari kecepatan jaringan)
- Mudah dalam instalasi
- Akurasi tinggi
  - Terutama Alpine Linux dan RHEL/CentOS
  - OS lainnya juga tetap memiliki akurasi tinggi
- DevSecOps
  - Cocok untuk CI seperti Travis CI, CircleCI, Jenkins, GitLab CI, etc.
  - [Contoh CI](https://aquasecurity.github.io/trivy/v0.22.0/advanced/integrations/)
- Mendukung Banyak Format
  - Container Images
    - Sebuah image lokal di Docker yang berjalan sebagai daemon
    - Sebeuah image lokal di Podman (>=2.0) yang expose ke socket
    - Sebuah remote image di docker registry seperti Docker Hub, ECR, GCR and ACR
    - Sebuah tar achive yang disimpan dengan format docker save atau podman
    - Sebuah direktori image yang sesuai dengan format [OCI Image](https://github.com/opencontainers/image-spec)
  - Local Filesystem dan Rootfs
  - Remote Git Repository

## Instalasi
Pada dasarnya untuk instalasi trivy bisa dijalankan pada beberapa sistem operasi, Binary, Helm, dan Docker

### RHEL/CentOS
Menambahkan repositori pada `/etc/yum.repos.d`
```bash
$ sudo vim /etc/yum.repos.d/trivy.repo
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/$releasever/$basearch/
gpgcheck=0
enabled=1
$ sudo yum -y update
$ sudo yum -y install trivy
```
atau jika mengunakan RPM
```
rpm -ivh https://github.com/aquasecurity/trivy/releases/download/v0.22.0/trivy_0.22.0_Linux-64bit.rpm
```

### Debian/Ubuntu
Menambahkan repositori pada `/etc/apt/sources.list.d`
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```
atau jika mengunakan DEB
```bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.22.0/trivy_0.22.0_Linux-64bit.deb
sudo dpkg -i trivy_0.22.0_Linux-64bit.deb
```

### Install Script
```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.22.0
```

### From source
```bash
mkdir -p $GOPATH/src/github.com/aquasecurity
cd $GOPATH/src/github.com/aquasecurity
git clone --depth 1 --branch v0.22.0 https://github.com/aquasecurity/trivy
cd trivy/cmd/trivy/
export GO111MODULE=on
go install
```
### Docker
```bash
docker pull aquasec/trivy:0.22.0
docker run --rm -v [YOUR_CACHE_DIR]:/root/.cache/ aquasec/trivy:0.22.0 [YOUR_IMAGE_NAME]
```

### Helm
```bash
helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/
helm repo update
helm search repo trivy
helm install my-trivy aquasecurity/trivy
```

## Quick Start
### Scan image container
untuk melakukan scan image container pada trivy sangat sederhana cukup masukan nama image dan tag
```bash
$ trivy image [YOUR_IMAGE_NAME]
```
Contoh
```bash
$ trivy image python:3.4-alpine
```
<details>
<summary>Result</summary>

```
2021-12-30T10:41:02.376+0700	INFO	Detected OS: alpine
2021-12-30T10:41:02.376+0700	INFO	Detecting Alpine vulnerabilities...
2021-12-30T10:41:02.378+0700	INFO	Number of language-specific files: 1
2021-12-30T10:41:02.378+0700	INFO	Detecting python-pkg vulnerabilities...
2021-12-30T10:41:02.379+0700	WARN	This OS version is no longer supported by the distribution: alpine 3.9.2
2021-12-30T10:41:02.379+0700	WARN	The vulnerability detection may be insufficient because security updates are not provided

python:3.4-alpine (alpine 3.9.2)
================================
Total: 37 (UNKNOWN: 0, LOW: 4, MEDIUM: 16, HIGH: 13, CRITICAL: 4)

+--------------+------------------+----------+-------------------+---------------+---------------------------------------+
|   LIBRARY    | VULNERABILITY ID | SEVERITY | INSTALLED VERSION | FIXED VERSION |                 TITLE                 |
+--------------+------------------+----------+-------------------+---------------+---------------------------------------+
| expat        | CVE-2018-20843   | HIGH     | 2.2.6-r0          | 2.2.7-r0      | expat: large number of                |
|              |                  |          |                   |               | colons in input makes parser          |
|              |                  |          |                   |               | consume high amount...                |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2018-20843 |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2019-15903   |          |                   | 2.2.7-r1      | expat: heap-based buffer              |
|              |                  |          |                   |               | over-read via crafted XML input       |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-15903 |
+--------------+------------------+----------+-------------------+---------------+---------------------------------------+
| libbz2       | CVE-2019-12900   | CRITICAL | 1.0.6-r6          | 1.0.6-r7      | bzip2: out-of-bounds write            |
|              |                  |          |                   |               | in function BZ2_decompress            |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-12900 |
+--------------+------------------+----------+-------------------+---------------+---------------------------------------+
| libcrypto1.1 | CVE-2019-1543    | HIGH     | 1.1.1a-r1         | 1.1.1b-r1     | openssl: ChaCha20-Poly1305            |
|              |                  |          |                   |               | with long nonces                      |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1543  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2020-1967    |          |                   | 1.1.1g-r0     | openssl: Segmentation                 |
|              |                  |          |                   |               | fault in SSL_check_chain              |
|              |                  |          |                   |               | causes denial of service              |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-1967  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-23840   |          |                   | 1.1.1j-r0     | openssl: integer                      |
|              |                  |          |                   |               | overflow in CipherUpdate              |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-23840 |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-3450    |          |                   | 1.1.1k-r0     | openssl: CA certificate check         |
|              |                  |          |                   |               | bypass with X509_V_FLAG_X509_STRICT   |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-3450  |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2019-1547    | MEDIUM   |                   | 1.1.1d-r0     | openssl: side-channel weak            |
|              |                  |          |                   |               | encryption vulnerability              |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1547  |
+              +------------------+          +                   +               +---------------------------------------+
|              | CVE-2019-1549    |          |                   |               | openssl: information                  |
|              |                  |          |                   |               | disclosure in fork()                  |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1549  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2019-1551    |          |                   | 1.1.1d-r2     | openssl: Integer overflow in RSAZ     |
|              |                  |          |                   |               | modular exponentiation on x86_64      |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1551  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2020-1971    |          |                   | 1.1.1i-r0     | openssl: EDIPARTYNAME                 |
|              |                  |          |                   |               | NULL pointer de-reference             |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-1971  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-23841   |          |                   | 1.1.1j-r0     | openssl: NULL pointer dereference     |
|              |                  |          |                   |               | in X509_issuer_and_serial_hash()      |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-23841 |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-3449    |          |                   | 1.1.1k-r0     | openssl: NULL pointer dereference     |
|              |                  |          |                   |               | in signature_algorithms processing    |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-3449  |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2019-1563    | LOW      |                   | 1.1.1d-r0     | openssl: information                  |
|              |                  |          |                   |               | disclosure in PKCS7_dataDecode        |
|              |                  |          |                   |               | and CMS_decrypt_set1_pkey             |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1563  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-23839   |          |                   | 1.1.1j-r0     | openssl: incorrect SSLv2              |
|              |                  |          |                   |               | rollback protection                   |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-23839 |
+--------------+------------------+----------+                   +---------------+---------------------------------------+
| libssl1.1    | CVE-2019-1543    | HIGH     |                   | 1.1.1b-r1     | openssl: ChaCha20-Poly1305            |
|              |                  |          |                   |               | with long nonces                      |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1543  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2020-1967    |          |                   | 1.1.1g-r0     | openssl: Segmentation                 |
|              |                  |          |                   |               | fault in SSL_check_chain              |
|              |                  |          |                   |               | causes denial of service              |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-1967  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-23840   |          |                   | 1.1.1j-r0     | openssl: integer                      |
|              |                  |          |                   |               | overflow in CipherUpdate              |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-23840 |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-3450    |          |                   | 1.1.1k-r0     | openssl: CA certificate check         |
|              |                  |          |                   |               | bypass with X509_V_FLAG_X509_STRICT   |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-3450  |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2019-1547    | MEDIUM   |                   | 1.1.1d-r0     | openssl: side-channel weak            |
|              |                  |          |                   |               | encryption vulnerability              |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1547  |
+              +------------------+          +                   +               +---------------------------------------+
|              | CVE-2019-1549    |          |                   |               | openssl: information                  |
|              |                  |          |                   |               | disclosure in fork()                  |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1549  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2019-1551    |          |                   | 1.1.1d-r2     | openssl: Integer overflow in RSAZ     |
|              |                  |          |                   |               | modular exponentiation on x86_64      |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1551  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2020-1971    |          |                   | 1.1.1i-r0     | openssl: EDIPARTYNAME                 |
|              |                  |          |                   |               | NULL pointer de-reference             |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-1971  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-23841   |          |                   | 1.1.1j-r0     | openssl: NULL pointer dereference     |
|              |                  |          |                   |               | in X509_issuer_and_serial_hash()      |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-23841 |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-3449    |          |                   | 1.1.1k-r0     | openssl: NULL pointer dereference     |
|              |                  |          |                   |               | in signature_algorithms processing    |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-3449  |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2019-1563    | LOW      |                   | 1.1.1d-r0     | openssl: information                  |
|              |                  |          |                   |               | disclosure in PKCS7_dataDecode        |
|              |                  |          |                   |               | and CMS_decrypt_set1_pkey             |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1563  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2021-23839   |          |                   | 1.1.1j-r0     | openssl: incorrect SSLv2              |
|              |                  |          |                   |               | rollback protection                   |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-23839 |
+--------------+------------------+----------+-------------------+---------------+---------------------------------------+
| musl         | CVE-2019-14697   | CRITICAL | 1.1.20-r4         | 1.1.20-r5     | musl libc through 1.1.23 has          |
|              |                  |          |                   |               | an x87 floating-point stack           |
|              |                  |          |                   |               | adjustment imbalance, related...      |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-14697 |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2020-28928   | MEDIUM   |                   | 1.1.20-r6     | In musl libc through 1.2.1,           |
|              |                  |          |                   |               | wcsnrtombs mishandles particular      |
|              |                  |          |                   |               | combinations of destination buffer... |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-28928 |
+--------------+------------------+----------+                   +---------------+---------------------------------------+
| musl-utils   | CVE-2019-14697   | CRITICAL |                   | 1.1.20-r5     | musl libc through 1.1.23 has          |
|              |                  |          |                   |               | an x87 floating-point stack           |
|              |                  |          |                   |               | adjustment imbalance, related...      |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-14697 |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2020-28928   | MEDIUM   |                   | 1.1.20-r6     | In musl libc through 1.2.1,           |
|              |                  |          |                   |               | wcsnrtombs mishandles particular      |
|              |                  |          |                   |               | combinations of destination buffer... |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-28928 |
+--------------+------------------+----------+-------------------+---------------+---------------------------------------+
| sqlite-libs  | CVE-2019-8457    | CRITICAL | 3.26.0-r3         | 3.28.0-r0     | sqlite: heap out-of-bound             |
|              |                  |          |                   |               | read in function rtreenode()          |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-8457  |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2019-19244   | HIGH     |                   | 3.28.0-r2     | sqlite: allows a crash                |
|              |                  |          |                   |               | if a sub-select uses both             |
|              |                  |          |                   |               | DISTINCT and window...                |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-19244 |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2019-5018    |          |                   | 3.28.0-r0     | sqlite: Use-after-free in             |
|              |                  |          |                   |               | window function leading               |
|              |                  |          |                   |               | to remote code execution              |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-5018  |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2020-11655   |          |                   | 3.28.0-r3     | sqlite: malformed window-function     |
|              |                  |          |                   |               | query leads to DoS                    |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-11655 |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2019-16168   | MEDIUM   |                   | 3.28.0-r1     | sqlite: Division by zero in           |
|              |                  |          |                   |               | whereLoopAddBtreeIndex in sqlite3.c   |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-16168 |
+              +------------------+          +                   +---------------+---------------------------------------+
|              | CVE-2019-19242   |          |                   | 3.28.0-r2     | sqlite: SQL injection in              |
|              |                  |          |                   |               | sqlite3ExprCodeTarget in expr.c       |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-19242 |
+--------------+------------------+----------+-------------------+---------------+---------------------------------------+

Python (python-pkg)
===================
Total: 4 (UNKNOWN: 1, LOW: 0, MEDIUM: 2, HIGH: 1, CRITICAL: 0)

+---------+------------------+----------+-------------------+---------------+---------------------------------------+
| LIBRARY | VULNERABILITY ID | SEVERITY | INSTALLED VERSION | FIXED VERSION |                 TITLE                 |
+---------+------------------+----------+-------------------+---------------+---------------------------------------+
| pip     | CVE-2019-20916   | HIGH     | 19.0.3            |          19.2 | python-pip: directory traversal       |
|         |                  |          |                   |               | in _download_http_url() function      |
|         |                  |          |                   |               | in src/pip/_internal/download.py      |
|         |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-20916 |
+         +------------------+----------+                   +---------------+---------------------------------------+
|         | CVE-2021-28363   | MEDIUM   |                   |          21.1 | python-urllib3: HTTPS proxy           |
|         |                  |          |                   |               | host name not validated when          |
|         |                  |          |                   |               | using default SSLContext              |
|         |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-28363 |
+         +------------------+          +                   +               +---------------------------------------+
|         | CVE-2021-3572    |          |                   |               | python-pip: Incorrect handling of     |
|         |                  |          |                   |               | unicode separators in git references  |
|         |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-3572  |
+         +------------------+----------+                   +               +---------------------------------------+
|         | pyup.io-42218    | UNKNOWN  |                   |               | Pip version 21.1 stops                |
|         |                  |          |                   |               | splitting on unicode                  |
|         |                  |          |                   |               | separators in git references,         |
|         |                  |          |                   |               | which...                              |
+---------+------------------+----------+-------------------+---------------+---------------------------------------+
```
</details>


Ref:
- https://aquasecurity.github.io/trivy/v0.22.0/