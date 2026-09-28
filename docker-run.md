# Docker run 

## Beispiel (binden an ein terminal), detached

```
# before that we did
docker pull ubuntu:24.04
docker run -t -d --name my_ubuntu ubuntu:24.04
# will wollen überprüfen, ob der container läuft
docker container ls 
# image vorhanden 
docker images

# in den Container reinwechsel 
docker exec -it my_ubuntu bash 
docker exec my_ubuntu cat /etc/os-release 
# 

```
