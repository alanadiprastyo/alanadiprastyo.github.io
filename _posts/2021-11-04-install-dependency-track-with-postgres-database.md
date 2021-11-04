# Install and Configure Dependency Track with Postgresql database

---
pada tulisan sebelum nya kita sudah membahas install dependency track, namun masih mengunakan database default H2 yang tidak disarankan digunakan pada env production

pada tulisan kali ini kita akan membahas install dependency track dengan database postgresql versi 13

sebelumnya kita perlu deploy database postgresql versi 13
```yaml
version: '3'

services:
  postgres:
    image: postgres:13.1
    healthcheck:
      test: [ "CMD", "pg_isready", "-q", "-d", "postgres", "-U", "root" ]
      timeout: 45s
      interval: 10s
      retries: 10
    restart: always
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=password
      - APP_DB_USER=dtrack
      - APP_DB_PASS=pass123
      - APP_DB_NAME=dtrack
    volumes:
      - ./db:/docker-entrypoint-initdb.d/
    ports:
      - 5432:5432
```
buat folder initial database
```bash
mkdir db

# vi db/01-init.sh
----
#!/bin/bash
set -e
export PGPASSWORD=$POSTGRES_PASSWORD;
psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
  CREATE USER $APP_DB_USER WITH PASSWORD '$APP_DB_PASS';
  CREATE DATABASE $APP_DB_NAME;
  GRANT ALL PRIVILEGES ON DATABASE $APP_DB_NAME TO $APP_DB_USER;
  \connect $APP_DB_NAME $APP_DB_USER
  BEGIN;
    CREATE TABLE IF NOT EXISTS event (
          id CHAR(26) NOT NULL CHECK (CHAR_LENGTH(id) = 26) PRIMARY KEY,
          aggregate_id CHAR(26) NOT NULL CHECK (CHAR_LENGTH(aggregate_id) = 26),
          event_data JSON NOT NULL,
          version INT,
          UNIQUE(aggregate_id, version)
        );
        CREATE INDEX idx_event_aggregate_id ON event (aggregate_id);
  COMMIT;
EOSQL
```
jalankan docker-compose postgresql dan pastikan contianernya berjalan dengan baik

kemudian download `docker-compose.yaml` pada dependency track dan modifikasi seperti dibawah ini

```bash
curl -LO https://dependencytrack.org/docker-compose.yml
```
update environment pada setingan database
```yaml
alpine.database.mode=external
alpine.database.url=jdbc:postgresql://IPADDRESS:5432/dtrack
alpine.database.driver=org.postgresql.Driver
alpine.database.username=dtrack
alpine.database.password=password
```
lebih detailnya seperti ini
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
    environment:
    # The Dependency-Track container can be configured using any of the
    # available configuration properties defined in:
    # https://docs.dependencytrack.org/getting-started/configuration/
    # All properties are upper case with periods replaced by underscores.
    #
    # Database Properties
     - ALPINE_DATABASE_MODE=external
     - ALPINE_DATABASE_URL=jdbc:postgresql://10.0.21.10:5432/dtrack
     - ALPINE_DATABASE_DRIVER=org.postgresql.Driver
     - ALPINE_DATABASE_USERNAME=dtrack
     - ALPINE_DATABASE_PASSWORD=pass123
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
          memory: 6192m
        reservations:
          memory: 4098m
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
      - API_BASE_URL=http://10.0.21.10:8081
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
jalankan deployment docker-compose dtrack

```bash
docker-compose up -d
```
cek log pada dtrack api untuk memastikan proses sinkronisasi repositori di internet selesai
```log
10:39:16.023 INFO [NpmAdvisoryMirrorTask] Retrieving NPM advisories from https://registry.npmjs.org/-/npm/v1/security/advisories?perPage=20&page=81
10:39:16.277 INFO [NpmAdvisoryMirrorTask] Updating datasource with NPM advisories
10:39:17.434 INFO [NpmAdvisoryMirrorTask] NPM advisory mirroring complete
10:44:22.848 INFO [NistMirrorTask] Initiating download of https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-2002.meta
10:44:24.170 INFO [NistMirrorTask] Downloading...
10:44:24.171 INFO [NistMirrorTask] Initiating download of https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-2003.json.gz
10:44:24.678 INFO [NistMirrorTask] Downloading...
10:44:25.698 INFO [NistMirrorTask] Uncompressing nvdcve-1.1-2003.json.gz
10:44:25.807 INFO [NvdParser] Parsing nvdcve-1.1-2003.json
10:46:16.148 INFO [NistMirrorTask] Initiating download of https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-2003.meta
10:46:17.453 INFO [NistMirrorTask] Downloading...
10:46:17.454 INFO [NistMirrorTask] Initiating download of https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-2004.json.gz
10:46:17.965 INFO [NistMirrorTask] Downloading...
10:46:19.011 INFO [NistMirrorTask] Uncompressing nvdcve-1.1-2004.json.gz
10:46:19.203 INFO [NvdParser] Parsing nvdcve-1.1-2004.json
10:49:49.037 INFO [NistMirrorTask] Initiating download of https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-2004.meta
10:49:50.358 INFO [NistMirrorTask] Downloading...
10:49:50.358 INFO [NistMirrorTask] Initiating download of https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-2005.json.gz
10:49:50.876 INFO [NistMirrorTask] Downloading...
10:49:53.442 INFO [NistMirrorTask] Uncompressing nvdcve-1.1-2005.json.gz
10:49:53.599 INFO [NvdParser] Parsing nvdcve-1.1-2005.json
10:55:54.799 INFO [NistMirrorTask] Initiating download of https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-2005.meta
10:55:56.135 INFO [NistMirrorTask] Downloading...
10:55:56.135 INFO [NistMirrorTask] Initiating download of https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-2006.json.gz
10:55:56.672 INFO [NistMirrorTask] Downloading...
10:55:58.318 INFO [NistMirrorTask] Uncompressing nvdcve-1.1-2006.json.gz
10:55:58.589 INFO [NvdParser] Parsing nvdcve-1.1-2006.json
```
atau pastikan tabel pada database dtrack di postgresql sudah terbentuk

setelah itu login ke dashboard `http://IP-ADDR:8080/`