1This is the process for configuring, accessing, and reading from a database using [[Openbao]].

Before touching anythin in openbao, a user in the database must be created, this will be the entry point of openbao for accessing the database.
# 1. Enable Database Secret Engine
For a dynamic use of database secrets, this user must be able to create other users. Openbao will create short lived users and manage their credentials on demand.

Once this user is created, we can proceed to openbao.

Inside it, go to "Secrets Engine" and enable a new engine for databases if non has been enabled before:

![[Pasted image 20260414092316.png]]![[Pasted image 20260414092333.png]]

# 2. DB connection inside DB Secrets Engine
Inside the new database secrets engine we can create new database connections. The process inside the UI is very straight forward. The only tricky part is the connection URL, which for variable values like username and password we must write as {{username}}. Example:
```
postgresql://{{username}}:{{password}}@192.168.1.149:5432/turnero?sslmode=disable
```

Now the connection has been completed, but we are not able to do anything with the database yet. To do any CRUD operation we must add a **role**

# 3. Bootstrap user (initial privileged DB user) - Credential Management

Openbao can create Dynamic or Static users that  will run queries specified in their **role**. 

A **role** is basically that, a set of queries specific to a single role. Meaning, we can create a role that creates users with only read capabilities.

Example:

Here is a role names **readonly** made for reading operations
![[Pasted image 20260414093948.png]]
![[Pasted image 20260414094001.png]]

The creation statements are the ones giving permission to the created user to read-only. This role is created as Dynamic, meaning, a user will only have a limited time of use before expiring. If expired, openbao will create a new one.

# 4. Authentication to Openbao from application (AppRole)
For an application to be able to access openbao and the desired secret engine. Just like a user, it needs a specific authentication method.

The recomended method is to use Approle, an authentication method used specifically for applications/services.

I will configure the authentication with Approle later on. At this point we only need to enable it. Either in CLI or the UI.

![[Pasted image 20260414104038.png]]

As with users, we must create an Entity for the application to use, it will be like its account (username/password).

But instead of using an username and password it has a **role_id** and a **secret_id**.

**role_id**: public-ish id that can be stored in plain text
**secret_id**: secret credential

These two are used to authenticate our app, once we give these two to openbao we will get a token. This token is then user to access secrets 

```bash
bao write auth/approle/login \
  role_id={{my_id}} \
  secret_id={{my_secret_id}}
```

# 5. AppRole Policy design
As with users, this Entity will have to permissions and its useless. For it to be able to access our database and read from it we need to assign it a policy.

In this case I will create a policy made only for this approle and reading-only. Following a **least-privilege policy** the policy should only:

- Read secrets from a scoped KV path
- Generate **read-only database credentials**
- Manage its own token lifecycle

#### **KV Secrets Access**:

```hcl
# vault-sample entity

# allows reading secrets under this endpoint
path "secret/data/vault-sample/*" {
	capabilities = [ "read" ]
}

# allows reading and listing metadata
path "secret/metadata/vault-sample/*" {
	capabilities = [ "read", "list" ]
}
```

`data/` -> actual secret values
`metadata/` -> versioning, listing

Without metadata access:
- You cannot list keys
- You must know exact paths

#### Dynamic Database Credentials (ESSENTIAL)

```hcl
path "database/creds/readonly" {
	capabilities = [ "read" ]
}
```

Triggers openbao to use the readonly role with this entity:

- Generate a temporary database user
-  Assign permissions defined in DB role **readonly**
- Return:
	- username
	- password
	- TTL

**NOTE**: this doesnot give direct DB access it only asks openbao to create DB credentials to then connect to DB

#### Additional KV PAth

```
path "secret/data/readonly" {
  capabilities = ["read"]
}
```

#### Token Self-Management

``` hcl
path "auth/token/renew-self" {
	capabilities = [ "update" ]
}

path "auth/token/lookup-self" {
	capabilities = [ "read" ]
}
```

- Renew its own token
-  Inspect its own token

# 6. Functional Flow
1. App authenticates via approle
2. Receives token with policy (vault-sample)
3.  App performs:
	1.  Fetch DB credentials from openbao -> `database/creds/readonly`
	2. Connect to database
	3. Read secrets if needed -> `secret/data/vault-sample/*`
	4. Maintain session -> `auth/token/renew-self`
# Summary

At this point openbao has been fully configured, we created:
- Database secrets engine
- A database connection to the secrets engine
- A custom role to the database with queries for user creation on readonly permission
- An authentication method for our application via AppRole
- Policies for this AppRole with access to the DB role for DB access

The next big step is to [[Connect an Application to Openbao]]

---
