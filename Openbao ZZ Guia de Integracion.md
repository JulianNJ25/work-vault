Server implementation of Openbao using it in a Docker image.
# 1. Install openbao in host machine

``` bash
sudo apt update
sudo apt isntall snapd
sudo snap install openbao

# crear shortcut para comando de openbao
echo "#alias para openbao" >> ~/.bashrc
echo "alias bao='openbao.bao'" >> ~/.bashrc

# check bao instalation
bao status
```

# 2. Install docker

Ubuntu guide: https://docs.docker.com/engine/install/ubuntu/

**Uninstall old versions**
``` bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```

**Set apt repository**
``` bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

**Install latest version**
``` shell
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Verity service**
``` bash
sudo systemctl status docker

# if no service is active
sudo systemctl start docker
sudo systectl enable docker
```

# 3. Get OpenBao docker image

URL: https://hub.docker.com/r/openbao/openbao

``` bash
docker pull openbao/openbao
```

# 4. Config OpenBao in host machine

**Create OpenBao directory structure for server and extra fucntionality**
```bash
 sudo mkdir -p /opt/openbao/{audit,config,data,logs,scripts,snapshots,tls}
```

**Create OpenBao config file**
``` bash
sudo touch /opt/openbao/openbao.hcl
```

Paste this into file:
Note -> do not forget to place the right IP server into {host_ip} placeholders
```
storage "raft" {
  path    = "/openbao/file"
  node_id = "node1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
#cluster_address = "127.0.0.1:8201"

  tls_cert_file = "/openbao/tls/bao.crt"
  tls_key_file  = "/openbao/tls/bao.key"
  tls_client_ca_file = "/openbao/tls/MyLocalCA.pem"
}

audit "file" "audit_log" {
  options = {
    file_path = "/openbao/audit/openbao_audit.log"
  }
}

#telemetry {
#  statsite_address = "127.0.0.1:8125"
#}

log_level = "info"

log_file = "/openbao/logs/openbao_server.log"

api_addr     = "https://tests.zuritalaboratorios.com:8200"
cluster_addr = "https://192.168.1.149:8201"


```

# 5. OpenBao CA certificate and TLS keys generation
**Bao cert generation config**
``` bash
sudo touch /opt/openbao/tls/bao.conf
sudo nano /opt/openbao/tls/bao.conf
cd /opt/openbao/tls/
```

Paste this into file
```TOML
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = v3_req

[dn]
C  = EC
ST = Pichincha
L  = Quito
O  = Zurita & Zurita Laboratorios
# Updated CN to the DNS entry
CN = tests.zuritalaboratorios.com

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
# DNS Entries
DNS.1 = tests.zuritalaboratorios.com
DNS.2 = localhost
DNS.3 = openbao

# IP Entries
IP.1 = 192.168.1.149
# internal docker container IP
IP.2 = 172.28.0.10
# Internal Docker Gateway
IP.3 = 172.28.0.1
IP.4 = 127.0.0.1
```

Run the next commands inside the /opt/openbao/tls directory

**Root CA key and self-signed certificate**
``` bash
# Generate CA Private Key
sudo openssl genrsa -out /opt/openbao/tls/MyLocalCA.key 4096

# Generate the Root certificate (self-signed)
sudo openssl req -x509 -new -nodes -key /opt/openbao/tls/MyLocalCA.key -sha256 -days 3650 -out /opt/openbao/tls/MyLocalCA.pem
```

**Bao server Entity Certificate (TLS certificate)**
```bash
# Generate a private key for OpenBao and a Certificate signing request
sudo openssl req -new -nodes -out /opt/openbao/tls/bao.csr -keyout /opt/openbao/tls/bao.key -config /opt/openbao/tls/bao.conf

# Sign certificate with the self-signed CA key
sudo openssl x509 -req -in /opt/openbao/tls/bao.csr -CA /opt/openbao/tls/MyLocalCA.pem -CAkey /opt/openbao/tls/MyLocalCA.key \
-CAcreateserial -out /opt/openbao/tls/bao.crt -days 825 -sha256 -extfile /opt/openbao/tls/bao.conf -extensions v3_req
```
**Note:** Modern browsers and systems often reject certificates with lifetimes longer than 825 days.

**Java Keystore & Truststore (PKCS12)**
```bash
# Combines theprivate key bao key and the signed CA certificate in a single file for the server to use
sudo openssl pkcs12 -export -out /opt/openbao/tls/bao-keystore.p12 -inkey /opt/openbao/tls/bao.key \
-in /opt/openbao/tls/bao.crt -certfile /opt/openbao/tls/MyLocalCA.pem -name "bao-server"
```

**Create truststore (.p12)**
```bash
# contains only the CA certificate, for clients to verify the server's certificate authenticity
sudo keytool -import -file /opt/openbao/tls/MyLocalCA.pem -alias "my-local-ca" \
-keystore /opt/openbao/tls/truststore.p12 -storetype PKCS12
```

**Add CA certificate to host machine internal ca-certificates**
```bash
sudo cp /opt/openbao/tls/MyLocalCA.pem /usr/local/share/ca-certificates/MyLocalCA.crt

sudo update-ca-certificates
```

**Files created**

| File             | Purpose                                                                               |
| ---------------- | ------------------------------------------------------------------------------------- |
| MyLocalCa.key    | **Critical Secret.** The private key of your CA.                                      |
| MyLocalCA.pem    | The public certificate of your CA (Install this on any client that will use openbao). |
| bao.key          | The private key for the OpenBao service.                                              |
| bao.crt          | The signed public certificate for OpenBao.                                            |
| bao-keystore.p12 | Bundled Key + Cert for the OpenBao listener.                                          |
| trustore.p12     | Contains the CA cert so applications know who to trust.                               |

# 6.  Docker compose config file

Create the following file and place it in the desired directory

`docker-compose.yml`
```YAML
services:
  openbao:
    image: quay.io/openbao/openbao:2.5.0
    container_name: openbao
    hostname: openbao

    user: "997:983"
    restart: unless-stopped

    networks:
      openbao-net:
        ipv4_address: 172.28.0.10

    ports:
      - "8200:8200"
      - "8201:8201"

    volumes:
      - /opt/openbao/config:/openbao/config:ro
      - /opt/openbao/data:/openbao/file
      - /opt/openbao/logs:/openbao/logs
      - /opt/openbao/tls:/openbao/tls:ro

    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m

    read_only: true

    cap_drop:
      - ALL
    cap_add:
      - IPC_LOCK

    security_opt:
      - no-new-privileges:true

    environment:
      - SKIP_CHOWN=true
      - VAULT_LOG_LEVEL=debug
      - BAO_DEV_ROOT_TOKEN_ID=
      - BAO_CACERT=/openbao/tls/MyLocalCA.pem  # Internal path
      - BAO_ADDR=https://tests.zuritalaboratorios.com:8200  

    command: ["bao", "server", "-config=/openbao/config"]

networks:
  openbao-net:
    driver: bridge
    name: openbao-net
    driver_opts:
      com.docker.network.bridge.name: "br-openbao"
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/24
          gateway: 172.28.0.1
```

# 7. Host user/group configuration and permissions 

URL: https://docs.docker.com/engine/install/linux-postinstall/

**Users/Groups**
``` bash
# create docker group
sudo groupadd docker
sudo groupadd openbao
# create openbao group and user
sudo useradd -u 997 -g 983 -s /usr/sbin/nologin -c "OpenBao service account" openbao

# assign openbao user to docker group
sudo usermod -aG docker openbao
```

**/opt/openbao/ permissions**

```bash
# -R flag recursively sets ownership to every file and sub-directory
sudo chown -R openbao:openbao /opt/openbao/

sudo chmod 644 /opt/openbao/tls/bao.conf /opt/openbao/tls/bao.crt /opt/openbao/tls/bao.csr /opt/openbao/tls/MyLocalCA.srl /opt/openbao/tls/truststore.p12

sudo chmod 600 /opt/openbao/tls/bao.key /opt/openbao/tls/bao-keystore.p12 /opt/openbao/tls/MyLocalCA.key

sudo chmod 750 /opt/openbao/config/openbao.hcl

sudo chmod 750 /opt/openbao/config /opt/openbao/data /opt/openbao/logs /opt/openbao/scripts /opt/openbao/snapshots /opt/openbao/tls 

sudo chmod 775 /opt/openbao/audit
```

# 8. Start Docker container and init Bao vault

Go to the location where docker-compose.yml was created and run
```bash
# run container
docker compose up -d

# check that container is properly running
docker compose logs openbao

# check that openbao server is up
bao status
```

Output for a properly running bao server
```
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            2.5.0
Build Date         2026-02-04T16:19:33Z
Storage Type       raft
HA Enabled         true
```

**Start vault**
```bash
bao operator init
```

Bao generates 5 unseal keys and 1 root key, we can unseld using at least 3 keys in any order

**Unseal vault**
```bash
# run this command 3 times
bao operator unseal
```

**Authenticate in browser**
```
https://192.168.1.149:8200
```

Use root token given with `bao operator init`

# 9. Configuration inside OpenBao Server
## 1. Create an ACL policy

Policies
![[Pasted image 20260423092642.png]]

Create ACL policy
![[Pasted image 20260423092713.png]]

Example of admin policy
```hcl
# =========================================
# ADMIN POLICY — OPENBAO (hardened)
# =========================================

# -------------------------
# SYSTEM
# -------------------------
path "sys/health" {
  capabilities = ["read"]
}
path "sys/capabilities" {
  capabilities = ["update"]
}
path "sys/capabilities-self" {
  capabilities = ["update"]
}
path "sys/capabilities-accessor" {
  capabilities = ["update"]
}
path "sys/internal/ui/*" {
  capabilities = ["read"]
}
path "sys/step-down" {
  capabilities = ["update", "sudo"]
}
path "sys/seal" {
  capabilities = ["update", "sudo"]
}
path "sys/unseal" {
  capabilities = ["update", "sudo"]
}
path "sys/rotate" {
  capabilities = ["update", "sudo"]
}

# -------------------------
# MOUNTS (SECRETS ENGINES)
# -------------------------
path "sys/mounts" {
  capabilities = ["read", "list"]
}
path "sys/mounts/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# -------------------------
# AUTH METHODS
# -------------------------
path "sys/auth" {
  capabilities = ["read", "list"]
}
path "sys/auth/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
path "auth/approle/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "auth/userpass/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# TOKENS
# -------------------------
path "auth/token/create" {
  capabilities = ["create", "update", "sudo"]
}
path "auth/token/create/*" {
  capabilities = ["create", "update", "sudo"]
}
path "auth/token/lookup" {
  capabilities = ["update"]
}
path "auth/token/lookup/*" {
  capabilities = ["read"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
path "auth/token/renew" {
  capabilities = ["update"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/revoke" {
  capabilities = ["update"]
}
path "auth/token/revoke-accessor" {
  capabilities = ["update"]
}
path "auth/token/revoke-self" {
  capabilities = ["update"]
}
path "auth/token/roles/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# POLICIES
# -------------------------
path "sys/policies/acl" {
  capabilities = ["list"]
}
path "sys/policies/acl/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# LEASES
# -------------------------
path "sys/leases/lookup" {
  capabilities = ["update"]
}
path "sys/leases/renew" {
  capabilities = ["update"]
}
path "sys/leases/revoke" {
  capabilities = ["update"]
}
path "sys/leases/revoke-prefix/*" {
  capabilities = ["update", "sudo"]
}
path "sys/leases/*" {
  capabilities = ["read", "update", "list"]
}

# -------------------------
# IDENTITY
# -------------------------
path "identity/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# KV SECRETS
# -------------------------
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# DATABASE (DYNAMIC CREDS)
# -------------------------
path "database/config/*" {
  capabilities = ["create", "read", "update", "delete"]
}
path "database/roles/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "database/creds/*" {
  capabilities = ["read"]
}

```
##  2. Set up user password authentication and Admin user

**Configure User - Password authentication**

On the left side menu select **Access**
![[Pasted image 20260422152259.png]]

Enable new method
![[Pasted image 20260422152319.png]]

Select Username & password
![[Pasted image 20260422152356.png]]

Enable method
![[Pasted image 20260422153004.png]]

**Create a user**
![[Pasted image 20260423101517.png]]

## 3. Create Group

Access
![[Pasted image 20260423102602.png]]

Groups
![[Pasted image 20260423102620.png]]

Assign desired policie\s, entities, and create group
![[Pasted image 20260423102737.png]]

## 4. Set Up Approle authentication

**Menu -> Access -> Authentication Methods -> Enable new method -> AppRole -> Create**
![[Pasted image 20260423103510.png]]

## 5. Database Connection and Configuration

### a. Secrets Engine for database:

**Menu -> Secret Engines -> Enable new engine -> Databases** -> Configure Engine -> **Enable engine**

### b. Connect to database:

**Menu -> Secrets Engine -> [database secrets engine]** -> **Connections** -> **Create connection**

**Connection URL must have this format for authentication via username and password**
```
postgresql://{{username}}:{{password}}@123.123.1.12:1111/turnero?sslmode=disable
```

![[Pasted image 20260423105231.png]]

Enable username template
![[Pasted image 20260423111059.png]]

### c. Create Role

**Menu -> Secrets Engine -> [database secrets engine]** -> **Roles** -> **Create role**

readonly role example:
![[Pasted image 20260423111247.png]]
![[Pasted image 20260423111257.png]]

```sql
# Creation statements
CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';

GRANT USAGE ON SCHEMA public TO "{{name}}";

GRANT SELECT ON ALL TABLES IN SCHEMA public TO "{{name}}";

ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO "{{name}}";
```

``` sql
# Revocation Statements
DROP ROLE IF EXISTS "{{name}}";
```

### d.  Create service policy to use readonly db role to talk to db
This policy will be attached to the Entity that the web service will use to authenticate to openbao an be able to talk to the DB 

**Menu -> Policies -> Create ACL policy**

Create policy
```
# =========================================
# POLICY — tunero-readonly-policy
# =========================================

# KV secrets — scoped to this project
path "secret/data/vault-sample/*" {
  capabilities = ["read"]
}
path "secret/metadata/vault-sample/*" {
  capabilities = ["read", "list"]
}

# Dynamic DB credentials — readonly role only
path "database/creds/readonly" {
  capabilities = ["read"]
}

path "database/creds/readonly" {
  capabilities = ["read"]
}

#KV engine
path "secret/data/readonly" {
  capabilities = ["read"]
}

# Token self-management
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
```

**Attach policy to the entity thats going to be used by webservice**
```bash
bao write auth/approle/role/vault-sample \ 
	token_policies="tunero-readonly-policy,default" \ 
	token_ttl=1h \ 
	token_max_ttl=4h \ 
	secret_id_ttl=24h \ 
	secret_id_num_uses=0 \
	# A valid SecretID is required in addition to the RoleID to authenticate. 
	bind_secret_id=true
```


# DEMO

command to add a JWT role with constrains of keycloak
```bash
cat <<EOF | bao write auth/jwt/role/db-service-role -
{
  "role_type": "jwt",
  "user_claim": "sub",
  "bound_issuer": "http://192.168.121.244:8080/realms/web-services",
  "bound_audiences": ["account"],
  "bound_claims": {
    "azp": "turnero-service"
  },
  "policies": ["vault-sample"],
  "ttl": "15m"
}
EOF
```