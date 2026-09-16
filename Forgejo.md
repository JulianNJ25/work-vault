
# Registrar una imagen docker

build the image

docker build -t [image name]] [path of docker file]

```bash
docker build -t localhost:3000/administrador/ci-maven21:ubuntu24 .
```

```
# 1. Log in to your host Docker CLI using localhost:3000
docker login localhost:3000 -u administrador

# 2. Re-tag your image with the exact target path
docker tag ci-maven21:ubuntu24 localhost:3000/administrador/ci-maven21:ubuntu24

# 3. Push to Forgejo's container registry
docker push localhost:3000/administrador/ci-maven21:ubuntu24
```
