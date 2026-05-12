[[Openbao]] has both userpass, approle, and token as a means of authentication
![[Pasted image 20260410093449.png]]
# **userpass**: human authentication

- username and password
- made for people
- allows login with CLI, UI
```bash
bao login -method=userpass username=admin
```

### Characteristics

- Long-lived identity
- Static credentials
- Tied to a human operator

# approle: machine authentication

- `role_id` (public identifier)
- `secret_id` (like a password)
```bash
bao write auth/approle/login \
    role_id=<role_id> \
    secret_id=<secret_id>
```
### Characteristics

- Non-interactive
- Ephemeral credentials
- Designed for automation

# Security implications

`userpass` for application generates security issues because using a **username** an a **password** imply:
- hardcoded credentials
- leaked `.env` / config files
-  no rotation of credentials
- eliminate static secrets

`approle` becomes better for services because of its two main authentication elements:

1. **role_id** (public-ish):
	1. can be embedded in the app
	2. not sensitive on its own
2. **secret_id** (sensitive):
	1. Delivered securely (CI/CD, init container, etc)
	2. Can be:
		1. time-limited
		2. one-time-use
		3. rotated

In a nutshell, **approle** has:
- No permanent credentials in code
- Credentials can expire
- Access can be tightly scoped
- Compromise window is small