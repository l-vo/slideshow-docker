class: impact

![Docker logo](images/docker.png)
  
# Let's discover Docker with some simple use cases

---

# Containers vs Virtual machines

![Containers versus virtual machines](images/containers-vms.jpg)

---
# Strengths and weaknesses
--

## Strengths
--

* User-friendly

--
* Ecosystem

--
* Need less memory than a VM

--
* Lightweight, start and stop quickly

--

## Weaknesses
--

* Less isolated than a VM

--
* Not designed for production

--
* Security

---

# Basic usage

Use a command which can be not present on our host system

```bash
$ docker run --rm debian:stretch sh -c 'echo -n "my string" | sha512sum'
54ee531987be4850329f3a8e65d12e00c02213856431a211a0eb328926e8273a7c30445f23ca96a7aa22ab90834f99d1a3308c34f4f89b601e811888c81d76b1
```
--
```bash
# Run cs fixer without having php installed on our host
$ docker run --rm  \
-v ~/Desktop/presentation-docker/cs-fixer/incomingTasks/:/wd \
shouldbee/php-cs-fixer fix /wd/tasks
```
---
# Basic usage

Log into a specific distribution for testing purpose

```bash
$ docker run -it --rm debian:stretch bash
$ docker run -it --rm debian:stretch # is equivalent for Debian Stretch image
root@8a0081aaf7b5:/# apt-get update && apt-get install -y nginx
```
---

# Building a container

## Dockerfile
```dockerfile
FROM debian:stretch

RUN apt-get update && apt-get -y install nginx

EXPOSE 80

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```
--
## Launch the build
```bash
$ docker build -t lvo/nginx .
```
---

# Building a container

## Docker images
```bash
$ docker images -a # Show all images included intermediate ones
REPOSITORY     TAG          IMAGE ID            CREATED        SIZE
lvo/nginx      latest       63ffd487e41f        13 hours ago   177MB
<none>         <none>       38d01827c618        13 hours ago   177MB
<none>         <none>       64bee46909e8        24 hours ago   177MB
debian         stretch      2b98c9851a37        5 weeks ago    100MB
```
--
## Remove a docker image
```bash
$ docker rmi 63ffd487e41f
$ docker rmi lvo/nginx # is equivalent
```

---

# Running a container
## Run the container
```bash
$ docker run -d --name nginx -p 80:80 lvo/nginx
```
--
## Log into the container
```bash
$ docker exec -it nginx bash
root@5efd51dbf09f:/#
```

---
# Managing a container

## Display container IP
```bash
$ docker inspect nginx # Display various information about the container
[{
    ...
    "NetworkSettings": {
        ...
        "Networks": {
            ...
            "IPAddress": "172.17.0.2",
            ....
        }
        ...
    }
    ...
}]
```
---
# Managing a container

## Display container IP
```bash
$ docker inspect -f \
'{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx
172.17.0.2
```
--
## Display active containers
```bash
$ docker ps
CONTAINER ID        IMAGE         COMMAND                  CREATED
       STATUS              PORTS                NAMES
5efd51dbf09f        lvo/nginx     "/usr/sbin/nginx -g …"   2 minutes ago
       Up 2 minutes        0.0.0.0:80->80/tcp   nginx
```
---
# Managing a container

## Stop and remove a container
```bash
$ docker stop 5efd51dbf09f
$ docker stop nginx # is equivalent
$ docker rm -vf 5efd51dbf09f
$ docker rm -vf nginx # is equivalent
```
---
# Managing a container

## Commit a container
```bash
$ docker run -it --rm debian:stretch bash
root@8a0081aaf7b5:/# ls -ld /etc/nginx
ls: cannot access '/etc/nginx': No such file or directory
root@8a0081aaf7b5:/# apt-get update && apt-get install -y nginx
root@8a0081aaf7b5:/# ls -ld /etc/nginx
drwxr-xr-x 8 root root 4096 Apr 22 15:05 /etc/nginx/
```
--
```bash
$ docker commit 8a0081aaf7b5 lvo/debian-nginx
sha256:697cc2a0ab9fe720617eb6208d689096c1bbc13f9daf6cea39899fda53d98e80
```
--
```bash
$ docker run -it --rm lvo/debian-nginx
root@aba59876bd61:/# ls -ld /etc/nginx
drwxr-xr-x 8 root root 4096 Apr 22 15:05 /etc/nginx/
```
---

# Using volumes

```dockerfile
FROM debian:stretch

RUN apt-get update && apt-get -y install nginx

VOLUME ["/var/www/html"]

EXPOSE 80

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

---

# Using volumes

## Named volumes
```bash
$ docker build -t lvo/nginx .
$ docker run -d --name nginx -p 80:80 \
-v nginx_files:/var/www/html lvo/nginx
```
--
### List named volumes

```bash
$ docker volume ls
DRIVER              VOLUME NAME
local               nginx_files
```
--
### Delete a named volume

```bash
$ docker volume rm nginx_files
```
---
# Using volumes

## Local files

```bash
$ docker run -d --name nginx -p 80:80 \
-v ~/Desktop/presentation-docker/nginx_volume/src:/var/www/html \
lvo/nginx
```

---
# Deploying a set of containers

## Modified nginx configuration file
```
# filename: default
# pass PHP scripts to FastCGI server
#
location ~ \.php$ {
	include snippets/fastcgi-php.conf;
	
	# With php-fpm (or other unix sockets):
	#fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
	# With php-cgi (or other tcp sockets):
	#fastcgi_pass 127.0.0.1:9000;
	fastcgi_pass phpfpm:9000;
}
```
---
# Deploying a set of containers

## Nginx Dockerfile
```dockerfile
FROM debian:stretch

RUN apt-get update && apt-get -y install nginx

COPY default /etc/nginx/sites-available

VOLUME ["/var/www/html"]

EXPOSE 80

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

---
# Deploying a set of containers

## docker-compose.yml
```
version: "3"
services:
  nginx:
    build: nginx # folder where the Dockerfile is
    ports:
      - 80:80
    volumes:
      - ./src:/var/www/html
  phpfpm: # Same name as in the nginx configuration file
    image: php:7.2-fpm-stretch
    volumes:
      - ./src:/var/www/html     
```


---
# Deploying a set of containers

## Run it !

```bash
# When you modify the nginx configuration file or Dockerfile
$ docker-compose build 
```
--
```bash
$ docker compose up -d
$ docker ps # dockercompose is the name of the folder
            # where the docker-compose.yml file is
CONTAINER ID        IMAGE                 NAMES
3b29cca150b8        php:7.2-fpm-stretch   dockercompose_phpfpm_1
b98111e647aa        dockercompose_nginx   dockercompose_nginx_1
```
---
# Going further
## Deploy containers on differents environments
### Environment variables
#### Command line
```bash
$ docker run --rm -e MY_VARIABLE=myvalue debian:stretch env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=a1fe43549ad5
MY_VARIABLE=myvalue
HOME=/root
```
---
# Going further
## Deploy containers on differents environments
### Environment variables
#### Docker-compose
```
version: '3'
  services:
  debian:
    image: debian:stretch
    environment:
      - MY_VARIABLE=myvalue
```
--
```bash
$ docker-compose run --rm debian env
# ...
MY_VARIABLE=myvalue
```
---
# Going further
## Deploy containers on differents environments
### Build arguments
```dockerfile
ARG debian_version
FROM debian:$debian_version
CMD ["cat", "/etc/os-release"]
```
--
#### Command line
```bash
$ docker build -t lvo/debian --build-arg debian_version=stretch .
$ docker run --rm lvo/debian
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
# ...
```
---
# Going further
## Deploy containers on differents environments
### Build arguments
#### Command line
```bash
$ docker build -t lvo/debian --build-arg debian_version=wheezy .
$ docker run --rm lvo/debian
PRETTY_NAME="Debian GNU/Linux 7 (wheezy)"
# ...
```
---
# Going further
## Deploy containers on differents environments
### Build arguments
#### Docker-compose
```
version: '3'
services:
  debian:
    build:
      context: .
      args:
        debian_version: jessie
```
--
```bash
$ docker-compose run --rm debian
PRETTY_NAME="Debian GNU/Linux 8 (jessie)"
# ...
```
---
# Going further
## Deploy containers on differents environments
### Docker-compose variable substitution
```
version: '3'
services:
  debian:
    build:
      context: .
      args:
        debian_version: ${DEBIAN_VERSION}
```
---
# Going further
## Deploy containers on differents environments
### Docker-compose variable substitution
#### Export variable
```bash
$ export DEBIAN_VERSION=stretch && docker-compose run --rm debian
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
# ...
```
---
# Going further
## Deploy containers on differents environments
### Docker-compose variable substitution
#### .env file
```
# .env
DEBIAN_VERSION=wheezy
```
--
```bash
$ docker-compose run --rm debian
PRETTY_NAME="Debian GNU/Linux 7 (wheezy)"
# ...
```
---
# Going further
## Deploy containers on differents environments
### Docker-compose override
```
# docker-compose.yml
version: '3'
services:
  nginx:
    image: nginx
    ports:
      - 80:80
```
--
```
$ docker-compose up -d && docker ps
CONTAINER ID        IMAGE               PORTS
23219ae555d2        nginx               0.0.0.0:80->80/tcp 
```
---
# Going further
## Deploy containers on differents environments
### Docker-compose override
```
# docker-compose.override.yml
version: '3'
services:
  nginx:
    ports:
      - 443:443
```
--
```
$ docker-compose up -d
$ docker ps
CONTAINER ID    IMAGE        PORTS
83f8087a7fcd    nginx        0.0.0.0:80->80/tcp,0.0.0.0:443->443/tcp
```
---
# Going further
## Deploy containers on differents environments
### Multiple docker-compose files
```
# docker-compose.yml
version: '3'
services:
  nginx:
    image: nginx
```
--
```
# docker-compose.dev.yml
version: '3'
services:
  nginx:
    ports:
      - 80:80
```
---
# Going further
## Deploy containers on differents environments
### Multiple docker-compose files
```
# docker-compose.prod.yml
version: '3'
services:
  nginx:
    ports:
      - 443:443
```
--
```bash
$ docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d
CONTAINER ID        IMAGE               PORTS
cf9bea982c29        nginx               0.0.0.0:80->80/tcp 
$ docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
CONTAINER ID        IMAGE               PORTS
1b7f06a1b094        nginx               80/tcp, 0.0.0.0:443->443/tcp
```
---
# Going further
## Security advices for production usage
* Use DOCKER_OPTS ***icc*** (change to false) and ***iptables*** (change to false too)

--
* Use DOCKER_OPTS ***userns-remap***

--
* When creating an image, use the ***USER*** directive when possible
---
class: impact
[https://github.com/l-vo/slideshow-docker](https://github.com/l-vo/slideshow-docker)
