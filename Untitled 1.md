docker login 192.168.1.194:3000 -u jnarvaez
docker build -t 192.168.1.194:3000/zurita/mkdocs-builder:1.0.0 .
docker push 192.168.1.194:3000/zurita/mkdocs-builder:1.0.0 