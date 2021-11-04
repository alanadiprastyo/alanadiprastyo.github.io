# Install and Configure Dependency Track use Docker Compose

---
Dependency Track (DT) adalah Platform Component Analysis yang mengizinkan suatu organisasi identifikasi dan menurunkan resiko pada sebuah proses development aplikasi.

Dependency Track mengunakan Software Bill of Materials (SBOM) dengan Cyclonedx yang berfungsi scanning semua dependency kemudian diexport menjadi file json yang nantinya akan di proses kedalam DT.


### **Deploying Docker Container**

**Prerequisite**
- install docker
- install docker-compose

untuk install Dependency Track terdapat dua service yaitu API Server dan Front End dengan spesifikasi sebagai berikut

- Container Requirements (API Server)

| Minimum 	| Recommended 	|
|---	|---	|
| 4.5GB RAM 	| 16GB RAM 	|
| 2 CPU cores 	| 4 CPU cores 	|

- Container Requirements (Front End)

| Minimum 	| Recommended 	|
|---	|---	|
| 512MB RAM 	| 1GB RAM 	|
| 1 CPU cores 	| 2 CPU cores 	|



- Setting Firewalld
```bash
# Downloads the latest Docker Compose file
curl -LO https://dependencytrack.org/docker-compose.yml
```
Note: Default database pada DT mengunakan H2 sehingga tidak disarankan mengunakan ini di production. untuk production DT mendukung beberapa database diantaranya adalah
1. Microsoft SQL Server 2012 dan diatasnya
2. MySQL 5.6 dan 5.7
3. PostgreSQL 9.0 and higher

kemudian edit bagian API_BASE_URL dan resource yang dibutuhkan pada docker-compose.yml

```yaml
version: '3.7'

#####################################################
# This Docker Compose file contains two services
#    Dependency-Track API Server
#    Dependency-Track FrontEnd
#####################################################

volumes:
  dependency-track:

services:
  dtrack-apiserver:
    image: dependencytrack/apiserver
    # environment:
    # The Dependency-Track container can be configured using any of the
    # available configuration properties defined in:
    # https://docs.dependencytrack.org/getting-started/configuration/
    # All properties are upper case with periods replaced by underscores.
    #
    # Database Properties
    # - ALPINE_DATABASE_MODE=external
    # - ALPINE_DATABASE_URL=jdbc:postgresql://postgres10:5432/dtrack
    # - ALPINE_DATABASE_DRIVER=org.postgresql.Driver
    # - ALPINE_DATABASE_USERNAME=dtrack
    # - ALPINE_DATABASE_PASSWORD=changeme
    # - ALPINE_DATABASE_POOL_ENABLED=true
    # - ALPINE_DATABASE_POOL_MAX_SIZE=20
    # - ALPINE_DATABASE_POOL_MIN_IDLE=10
    # - ALPINE_DATABASE_POOL_IDLE_TIMEOUT=300000
    # - ALPINE_DATABASE_POOL_MAX_LIFETIME=600000
    #
    # Optional LDAP Properties
    # - ALPINE_LDAP_ENABLED=true
    # - ALPINE_LDAP_SERVER_URL=ldap://ldap.example.com:389
    # - ALPINE_LDAP_BASEDN=dc=example,dc=com
    # - ALPINE_LDAP_SECURITY_AUTH=simple
    # - ALPINE_LDAP_BIND_USERNAME=
    # - ALPINE_LDAP_BIND_PASSWORD=
    # - ALPINE_LDAP_AUTH_USERNAME_FORMAT=%s@example.com
    # - ALPINE_LDAP_ATTRIBUTE_NAME=userPrincipalName
    # - ALPINE_LDAP_ATTRIBUTE_MAIL=mail
    # - ALPINE_LDAP_GROUPS_FILTER=(&(objectClass=group)(objectCategory=Group))
    # - ALPINE_LDAP_USER_GROUPS_FILTER=(member:1.2.840.113556.1.4.1941:={USER_DN})
    # - ALPINE_LDAP_GROUPS_SEARCH_FILTER=(&(objectClass=group)(objectCategory=Group)(cn=*{SEARCH_TERM}*))
    # - ALPINE_LDAP_USERS_SEARCH_FILTER=(&(objectClass=user)(objectCategory=Person)(cn=*{SEARCH_TERM}*))
    # - ALPINE_LDAP_USER_PROVISIONING=false
    # - ALPINE_LDAP_TEAM_SYNCHRONIZATION=false
    #
    # Optional OpenID Connect (OIDC) Properties
    # - ALPINE_OIDC_ENABLED=true
    # - ALPINE_OIDC_ISSUER=https://auth.example.com/auth/realms/example
    # - ALPINE_OIDC_USERNAME_CLAIM=preferred_username
    # - ALPINE_OIDC_TEAMS_CLAIM=groups
    # - ALPINE_OIDC_USER_PROVISIONING=true
    # - ALPINE_OIDC_TEAM_SYNCHRONIZATION=true
    #
    # Optional HTTP Proxy Settings
    # - ALPINE_HTTP_PROXY_ADDRESS=proxy.example.com
    # - ALPINE_HTTP_PROXY_PORT=8888
    # - ALPINE_HTTP_PROXY_USERNAME=
    # - ALPINE_HTTP_PROXY_PASSWORD=
    # - ALPINE_NO_PROXY=
    #
    # Optional Cross-Origin Resource Sharing (CORS) Headers
    # - ALPINE_CORS_ENABLED=true
    # - ALPINE_CORS_ALLOW_ORIGIN=*
    # - ALPINE_CORS_ALLOW_METHODS=GET POST PUT DELETE OPTIONS
    # - ALPINE_CORS_ALLOW_HEADERS=Origin, Content-Type, Authorization, X-Requested-With, Content-Length, Accept, Origin, X-Api-Key, X-Total-Count, *
    # - ALPINE_CORS_EXPOSE_HEADERS=Origin, Content-Type, Authorization, X-Requested-With, Content-Length, Accept, Origin, X-Api-Key, X-Total-Count
    # - ALPINE_CORS_ALLOW_CREDENTIALS=true
    # - ALPINE_CORS_MAX_AGE=3600
    deploy:
      resources:
        limits:
          memory: 12288m
        reservations:
          memory: 8192m
      restart_policy:
        condition: on-failure
    ports:
      - '8081:8080'
    volumes:
      - 'dependency-track:/data'
    restart: unless-stopped

  dtrack-frontend:
    image: dependencytrack/frontend
    depends_on:
      - dtrack-apiserver
    environment:
      # The base URL of the API server.
      # NOTE:
      #   * This URL must be reachable by the browsers of your users.
      #   * The frontend container itself does NOT communicate with the API server directly, it just serves static files.
      #   * When deploying to dedicated servers, please use the external IP or domain of the API server.
      - API_BASE_URL=http://IP-ADDR:8081
      # - "OIDC_ISSUER="
      # - "OIDC_CLIENT_ID="
      # - "OIDC_SCOPE="
      # - "OIDC_FLOW="
      # - "OIDC_LOGIN_BUTTON_TEXT="
      # volumes:
      # - "/host/path/to/config.json:/app/static/config.json"
    ports:
      - "8080:8080"
    restart: unless-stopped

```

setelah itu login ke dashboard `http://IP-ADDR:8080` dengan credential default seperti dibawah ini
- username : admin
- password : admin

kemudian seting analyzers untuk menambah mesin analisa, sehingga nantinya hasil temuan dari SBOM akan semakin relevan, untuk saat ini hanya ada 4 yaitu :
1. Internal (National Vulnerability Database)
2. NPM Audit
3. Sonatype OSS Index (free, tapi harus registrasi dulu)
4. VulnDB (Berbayar)

**Scan Dependencies dengan Cyclonedx**

pada tulisan ini saya mengunakan repo https://github.com/alanadiprastyo/demo-devsecops yang berbasis python

Cyclonedx merupakan sebuah tools yang berfungsi untuk scan dependency menjadi format SBOM

- install cyclonedx
```
sudo pip3 install cyclonedx-bom
```

- jalankan cyclonedx-py 
```
cyclonedx-py -r -i requirements.txt --format json --schema-version 1.3 -o sbom.json
```

**Membuat Project**

untuk memulai mengunakan DT kita perlu membuat project untuk mengupload file json SBOM kemudian akan dianalisa oleh DT

Project > Create Project

kemudian isikan Project Name dan Classifier (jenis aplikasi yang discan apakah Aplication, Framework, Library atau lainnya)

setelah membuat project upload file json SBOM melalui web frontend atau melalui rest api, setelah di upload DT akan melakukan analisa dan menampilkan report seperti ini

![project](/img/dtrack/dash-project.png)

pada library Jinja2 terdapat temuan seperti gambar dibawah ini
![project](/img/dtrack/detect-jinja2.png)

Ref:
- https://docs.dependencytrack.org/getting-started/deploy-docker/
- https://cyclonedx.org/