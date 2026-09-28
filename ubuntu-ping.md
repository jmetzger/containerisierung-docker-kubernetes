# Ubuntu mit ping 

```
cd 
mkdir myubuntu 
cd myubuntu/
```

```
# nano Dockerfile
FROM ubuntu:24.04
RUN apt-get update && \
    apt-get install -y inetutils-ping && \
    rm -rf /var/lib/apt/lists/*
# CMD ["/bin/bash"]
```


```
docker build -t myubuntu:24.04-ping .
docker images
# -t wird benötigt, damit bash WEITER im Hintergrund im läuft.
# auch mit -d (ohne -t) wird die bash ausgeführt, aber "das Terminal" dann direkt beendet 
# -> container läuft dann nicht mehr 
docker run -d -t --name container-ubuntu myubuntu:24.04-ping
docker container ls
# in den container reingehen mit dem namen des Containers: container-ubuntu 
docker exec -it container-ubuntu bash
ls -la
```

```
docker inspect container-ubuntu
docker inspect container-ubuntu | grep -i "ip"
```

```
# Zweiten Container starten
docker run -d -t --name container-ubuntu2 myubuntu:24.04-ping

# Ersten Container -> 2. anpingen 
docker exec -it container-ubuntu2 bash 
# jetzt den container-ubuntu anpingen 
ping -c4 <ip-von-container-ubuntu>
```
