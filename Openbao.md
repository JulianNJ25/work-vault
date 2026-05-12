
# Quick Production Checklist

| Step                                       | Done? |
| ------------------------------------------ | ----- |
| Integrated Storage (Raft) with 3+ nodes    | ☐     |
| TLS on all listeners                       | ☐     |
| Auto Unseal configured (KMS/HSM)           | ☐     |
| Root token revoked after init              | ☐     |
| Recovery keys stored securely (PGP)        | ☐     |
| At least 2 audit devices enabled           | ☐     |
| Auth methods set up (no direct root usage) | ☐     |
| Least-privilege policies per service       | ☐     |
| Secret engines mounted as needed           | ☐     |
| OpenBao Agent deployed alongside apps      | ☐     |
| Monitoring on `/v1/sys/health`             | ☐     |
| Backup strategy for Raft snapshots         | ☐     |

# Installation and Configuration Guide

# 1. Installation

If docker is not installed, check for an ubuntu machine: https://docs.docker.com/engine/install/ubuntu/

Installation of OpenBao via docker, see: https://openbao.org/downloads/
```bash
docker pull quay.io/openbao/openbao:2.5.2
```

# 2. OpenBao config file

Create openbao directory and add config file
```
mkdir -p /etc/openbao/config

touch /etc/openbao/config/openbao.hcl
```

Inside openbao.hcl paste this config:
```
storage "raft" {
  path    = "/openbao/data"
  node_id = "node1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"

  tls_cert_file = "/openbao/tls/bao.crt"
  tls_key_file  = "/openbao/tls/bao.key"


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

log_file = "/openbao/log/openbao_server.log"

api_addr     = "https://[server_ip]:8200"
cluster_addr = "https://[server_ip]:8201"

ui            = true
cluster_name  = "prod-openbao"

# needed for docker
disable_mlock = true

default_lease_ttl = "24h"
max_lease_ttl     = "720h"

```




baseline config:
```htl
storage "raft" {
  path    = "/opt/openbao/data"
  node_id = "node1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/openbao/tls/fullchain.pem"
  tls_key_file  = "/opt/openbao/tls/privkey.pem"
}

# auditing is disable on cli by default, it must be defined here if wanted
audit "file" "audit_log" {  
  options = {   
	file_path = "/var/log/openbao/audit.log"  
  }  
}

api_addr     = "https://bao1.internal.yourcompany.com:8200"
cluster_addr = "https://bao1.internal.yourcompany.com:8201"

ui            = true
cluster_name  = "prod-openbao"

default_lease_ttl = "24h"
max_lease_ttl     = "720h"
```

I generated my own tls keys and certs using:
```
openssl req -x509 -nodes -days 365 \
  -newkey rsa:4096 \
  -keyout ~/tls/bao.key \
  -out ~/tls/bao.crt \
  -config ~/tls/openssl.cnf \
  -extensions req_ext
```

To ignore the "handmade" certificate this environmental variable is set:
```bash
export BAO_SKIP_VERIFY=true
```

After the configuration is written, use the `-config` flag with `bao server` to specify where the configuration is.

In my case I had 2 issues, the second issue is addressed in the next general step:

1.- Starting server 

I had to find a workaround since Im in vagran, with access to only one shell. To initiate the bao server in the bakground and send all the logs I run this command

```bash
nohup bao server -config /etc/openbao/openbao.hcl > bao.log 2>&1 &
```

I created a bash file to automate this process and have a more user friendly output

```bash
#!/bin/bash

# Set OpenBao address (important for CLI later)
export VAULT_ADDR="http://127.0.0.1:8200"

# Start OpenBao in background
nohup bao server -config /etc/openbao/openbao.hcl > /var/log/openbao.log 2>&1 &

# Save PID
echo $! > openbao.pid

echo "OpenBao started with PID $(cat openbao.pid)"
```
**What each part does**

- `nohup` → keeps it running after logout
- `> ... 2>&1` → redirects logs (stdout + stderr)
- `&` → runs in background
- `$!` → gets the process ID (PID)

and for stopping

```bash
#!/bin/bash

if [ -f openbao.pid ]; then
  kill $(cat openbao.pid)
  rm openbao.pid
  echo "OpenBao stopped"
else
  echo "No PID file found"
fi
```

At this point:
- OpenBao server has **no data**
- It is **sealed**
- It cannot store or return secrets

## Integrated Storage (Raft) with 3+ nodes

After starting the server and running the next command

```
bao operator init
```

This command causes:

- The storage backend (Raft) is **bootstrapped**
- Encryption keys are generated [ master /  unseal / root ]
- The system becomes **ready to be unsealed and used**

```
Get "https://127.0.0.1:8200/v1/sys/seal-status": dial tcp 127.0.0.1:8200: connect: connection refused 
```

In the config file I disabled TLS, yet openbao is trying to use https. To solve this the following path variable must be set

``` bash
export VAULT_ADDR="http://127.0.0.1:8200"
```

This problem was already solved in the init bash file for the server.

After successfully generating all the keys and root token.  

```
Unseal Key 1: nMCNyuZ9cATq9885EQwytE0b+KW++1yqBr24Dd3CfED+
Unseal Key 2: CuMz6Jii55BERDEAy6cC9CyzLN7QHNWDjCIVFw9MX+yo
Unseal Key 3: ljFxk0tyBA5RLLR9UuRxA9JO8SIxcwd5Jrq6CkIImu07
Unseal Key 4: zxt5DJrvaDLapM1tgDF/DCmBcAl/CyEjsCasy0Q4tGCG
Unseal Key 5: dXbdAk2PJw8u6xqratVNcLbx3OWThReMgYIGIvjAzLlC

Initial Root Token: s.itnDU21yTj7XHMaQ1ZKdPaod
```

The next step is to unseal the vault with:
```
bao operator unseal
```
 I run this command 3 times, as that is the minimum to **unseal** the vault
[^1]

[^1]: Note:
	In a production setting, it would be ideal to have multiple instances of openbao servers (minimun 3). To do this run `bao operator raft join http://<node1-ip>:8200`

Lastly, 

**Audit and logs**:

Its recommended to enable auditing and system login for permanent storage of logs

This was already done in the config with:

```
# auditing is disable on cli by default, it must be defined here if wanted
audit "file" "audit_log" {  
  options = {   
	file_path = "/var/log/openbao/audit.log"  
  }  
}
```

This I a command that can be used to enable auditing
```bash
bao audit enable file file_path=/var/log/openbao/audit.log
bao audit enable syslog
```

However I ran into some issues. First:

```bash
Error enabling audit device: Put "https://127.0.0.1:8200/v1/sys/audit/file": tls: failed to verify certificate: x509: certificate relies on legacy Common Name field, use SANs instead
```

This was caused since sudo was ignoring all environmental variables so `export BAO_SKIP_VERIFY=true` was not being read. To solve this, we have do change permissions on the /var/log/openbao directory

```bash
sudo mkdir -p /var/log/openbao
sudo chown $USER:$USER /var/log/openbao
```

Ultimately I whent with a file inside /tmp since this is alway writable and does not require to change permisions

Next step is to set this varible in the server, since API/CLI auditing is not enabled by default:

```
`unsafe_allow_api_audit_creation = true`
```

After this try to run the `bao audit enable [path]` command

Now OpenBao has been configured to allow use.

# OpenBao Post config
Now we can log in with the root token, it would be ideal to create different users so we dont only use root.

For this a new policy for each user must be created. Then create the user and assign the policy

Initially I created an [[ openbao admin]] user (identity), to make it act as an admin we have to create an admin policy and assign it to it

```hcl
# =========================================

# ADMIN POLICY — OPENBAO

# =========================================

# -------------------------

# SYSTEM

# -------------------------

path "sys/health" {
capabilities = ["read"]
}

path "sys/capabilities-self" {
capabilities = ["update"]
}

path "sys/internal/ui/*" {
capabilities = ["read"]
}

# -------------------------

# MOUNTS (SECRETS ENGINES)

# -------------------------

path "sys/mounts" {
capabilities = ["read", "list"]
}

path "sys/mounts/*" {
capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------

# AUTH METHODS

# -------------------------

path "sys/auth" {
capabilities = ["read", "list"]
}

path "sys/auth/*" {
capabilities = ["create", "read", "update", "delete", "list"]
}

# AppRole management

path "auth/approle/*" {
capabilities = ["create", "read", "update", "delete", "list"]
}

# Userpass management (optional but useful)

path "auth/userpass/*" {
capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------

# TOKENS

# -------------------------

path "auth/token/*" {
capabilities = ["create", "read", "update", "delete", "list"]
}

path "auth/token/lookup/*" {
capabilities = ["read"]
}

path "auth/token/lookup-self" {
capabilities = ["read"]
}

path "auth/token/renew-self" {
capabilities = ["update"]
}

path "auth/token/revoke-self" {
capabilities = ["update"]
}

# -------------------------

# POLICIES

# -------------------------

path "sys/policies/acl/*" {
capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------

# LEASES

# -------------------------

path "sys/leases/*" {
capabilities = ["read", "update", "list"]
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

path "database/*" {
capabilities = ["read", "list"]
}

path "database/config/*" {
capabilities = ["create", "read", "update", "delete"]
}

path "database/roles/*" {
capabilities = ["create", "read", "update", "delete"]
}

path "database/creds/*" {
capabilities = ["read"]
}


```

``` bash
#Confirm your userpass user has it assigned 
bao read auth/userpass/users/YOUR_USERNAME

#If policy isn't listed there, update the user 
bao write auth/userpass/users/YOUR_USERNAME \ token_policies="admin,default"
```
we must also assign this policy as a token policy to be able to manage tokens

In this project I will be connecting to a company development database.

I found an issue when trying to connect to the database through the UI
```
failed to retrieve username_template: invalid value at username_template: is a <nil>
```

To solve this a user template create must be created. But this can only be defined inside the CLI or the API.
Ultimately I used this command to create the database through the CLI
[^2]

```bash
bao write database/config/my-postgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="readonly" \
  connection_url="postgresql://{{username}}:{{password}}@192.168.1.149:5432/turnero?sslmode=disable" \
  username="YOUR_DB_USER" \
  password="YOUR_DB_PASSWORD" \
  username_template="v-{{random 8}}"
```

For this, an openbao admin role specific for creating users should be created in the database, and use its credentials as shown above.

[^2]: To be able to write to the database/ endpoint we must give our user database access in the policy, as shown in the admin policy above, else it won't work

After succesfully connecting to the DB, the next step is to create roles for the automatization of the creation of db users.

For example, I created an role specific for read-only users:
![[Pasted image 20260409122846.png]]

```sql
# Creation statements
CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';

GRANT USAGE ON SCHEMA public TO "{{name}}";

GRANT SELECT ON ALL TABLES IN SCHEMA public TO "{{name}}";

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO "{{name}}";
```

``` sql
# Revocation Statements
DROP ROLE IF EXISTS "{{name}}";
```

# Approle

To properly connect the db with openbao, an approle must be created in order to ensure security is on top.

Approle as mentioned in [[userpass vs approle]] is an auth method for applications and services.

As with any other auth method a policy must be applied to it. 

For this project the policy **vault-sample-policy.hct** was created:

```hcl
# =========================================
# POLICY — vault-sample (zuritalaboratorios)
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

# Token self-management
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}

```

```bash
bao policy write vault-sample /path/to/vault-sample-policy.hcl
```

then the approle **vault-sample** was created and assigned this pollicy
```bash
bao write auth/approle/role/vault-sample \ 
	token_policies="vault-sample,default" \ 
	token_ttl=1h \ 
	token_max_ttl=4h \ 
	secret_id_ttl=24h \ 
	secret_id_num_uses=0 \
	# A valid SecretID is required in addition to the RoleID to authenticate. 
	bind_secret_id=true
```

**approles** have two auth fields:
- roleID: public identifier
- secretID: confidential value (password-like)

In the CLI command above, **bind_secret_id** was declared s openbao needs it for authentication, meaning when openbao will request these fields:

```json
{
  "role_id": "...",
  "secret_id": "..."
}
```

instead of only:
```json
# very insecure
{
  "role_id": "...",
}
```

To get those values we can run:

```bash
# 4. Get RoleID 
bao read auth/approle/role/vault-sample/role-id 

# 5. Generate SecretID 
bao write -f auth/approle/role/vault-sample/secret-id
```

Note tha the secret-id id like the root token, it cannot be read, only generated, so we must save this properly



