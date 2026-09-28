# Docker Compose example Wordpress / MySQL 

## Walkthrough 

```
clear
cd
mkdir wp
cd wp
nano docker-compose.yaml
```

```
services:
  database:
    image: mysql:5.7
    volumes:
      - database_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: mypassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress

  wordpress:
    image: wordpress:latest
    depends_on: # wartet bis database gestartet ist. NICHT ob sie bereit ist 
      - database
    ports:
      - 8080:80
    restart: always
    environment:
      WORDPRESS_DB_HOST: database:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
    volumes:
      - wordpress_plugins:/var/www/html/wp-content/plugins
      - wordpress_themes:/var/www/html/wp-content/themes
      - wordpress_uploads:/var/www/html/wp-content/uploads

volumes:
  database_data:
  wordpress_plugins:
  wordpress_themes:
  wordpress_uploads:


```


```
docker compose version
docker compose up -d  # alle services im docker-compose.yaml starten und im
                      # Hintergrund laufen lassen
docker compose logs   # Alle Logs dieses Projektes
docker compose ps     # alle container die zu diesem Projekt gehören
docker compose down
# bitte keine start und stop -> immer stattdessen up und down
```

```
# Im Browser mit ip des server -> ip a show eth0 # hier die externe ip raussuchen
http://<ip-des-servers>:8080
```

## Healthcheck


```
nano docker-compose.yaml
# CTRL/STRG + K -> alte Fassung rausloeschen 
```

```
# docker-compose.yaml

services:
  database:
    image: mysql:5.7
    volumes:
      - database_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: mypassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -uroot -p$$MYSQL_ROOT_PASSWORD"]
      interval: 5s
      timeout: 3s
      retries: 10

  wordpress:
    image: wordpress:latest
    depends_on:
      database:
        condition: service_healthy   # wartet, bis database bereit ist
    ports:
      - 8080:80
    restart: always
    environment:
      WORDPRESS_DB_HOST: database:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
    healthcheck:                     # prüft, ob WordPress die DB erreicht
      test: ["CMD-SHELL", "php -r '$$c = @new mysqli(\"database\", getenv(\"WORDPRESS_DB_USER\"), getenv(\"WORDPRESS_DB_PASSWORD\")); exit($$c->connect_errno ? 1 : 0);'"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    volumes:
      - wordpress_plugins:/var/www/html/wp-content/plugins
      - wordpress_themes:/var/www/html/wp-content/themes
      - wordpress_uploads:/var/www/html/wp-content/uploads

volumes:
  database_data:
  wordpress_plugins:
  wordpress_themes:
  wordpress_uploads:

```
