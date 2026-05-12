Credentials can be in many files:
- git
- jira
- helpbooks
- etc
# Elements

## Why use Vault

### Discovery
We can use Vault discovery to find all secrets used in our systems

### Vault
We can also audit so we can know who access what secret

Roll-base access control to manage the minimum need of access to secrets:
Example: 
- If its needed that both a program and a user access those keys, - least privilege

Encryption of secrets and centralized storage for them

### Auto-rotation
Sets a rotation time for secrets so they are not long lived

### Dynamic secrets
Short lived authentication, so a credential is unique to each instance