
### Step 1 — Push the image as a container package to Forgejo

**1. Create a token with package write access**  
In Forgejo: Settings → Applications → Generate New Token, with `write:package` (and `read:package`) scope. If the image will live under an org, make sure your account has package-write permission there.

**2. Log in to the registry**

bash

```bash
docker login 192.168.1.194:3000 -u jnarvaez
# paste the token when prompted for the password
```

**3. Build and tag the image**  
Forgejo's registry namespace pattern is `<domain>/<owner>/<package-name>:<tag>`, where `<owner>` is the user or org that will own the package:

bash

```bash
docker build -t 192.168.1.194:3000/zurita/[name]:[version] .

docker build -t 192.168.1.194:3000/zurita/java21-maven:1.0.0 .
```

**4. Push it**

bash

```bash
docker push 192.168.1.194:3000/zurita/[name]:[version]
```