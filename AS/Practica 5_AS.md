Descargamos los contenedores de `Vulhub`:
```
git clone https://github.com/vulhub/vulhub.git
```

levantamos la máquina de `docker`:
```
cd vulhub/httpd/CVE-2021-41773
docker compose up -d
```
scaneamos la red para descubrir su ip