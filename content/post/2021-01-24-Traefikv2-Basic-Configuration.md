---
title: "Traefik v2: Basic handson and configuration for TLS and basic_auth"
date: "2021-01-24"
tags:
- docker
- docker-compose
- traefik
- reverse-proxy
categories:
- blog
- sysadmin
- docker
- selfhosted
author: jtorrex
---

# Description:

In this post I will describe how to use Traefik as a reverse proxy to expose diferent services with autogenerated certificates and basic auth.

# Traefik: Basic example

The idea was to deploy Traefik using Docker, configuring the rest of the containerized services using docker-compose to manage them.

* Install the requisites to have docker and docker-compose installed on your machine. This procedure varies depending on your operating system.

* Create a docker-compose.yml file the following content.

```version: "3.3"

services:
  traefik:
    image: "traefik:v2.4"
    container_name: "traefik"
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
	- "--entrypoints.websecure.address=:80"
    ports:
      - "80:80"
	- "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

  whoami:
    image: "containous/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"``
```

When this docker-compose.yml was executed, it pulls two images (traefik and a basic webservice), applying all the options included on the defined services.

To customize this test, replace whoami.localhost by your own domain, obtaining that this basic service was reachable from the external net of the machine.

* On the same directory where the docker-compose.yml file reside, run:

```docker-compose up -d
```

This start the dockerized stack that you defined before.

When the stack was running (you can check it with):

```watch docker ps
```

Visit your defined domain to check if the webservice was correctly published, you should see something like:

```
Hostname: 48eaa1dasb9
IP: 127.0.0.1
IP: 172.20.0.3
RemoteAddr: 160.20.0.5:46872
GET / HTTP/1.1
Host: whoami.torrex.xyz
User-Agent: Mozilla/4.0 (X11; Linux x86; rv:84.0) Gecko/20100101 Firefox/83.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip
Accept-Language: en-US,en;q=0.5
Cf-Connecting-Ip: XXX.XXX.XXX.XXX
Cf-Ipcountry: US
Cf-Ray: 234502df442342
Cf-Request-Id: 07d7b2abd1023f34r54f
Cf-Visitor: {"scheme":"https"}
Cookie: __cfduid=d4f2d123453245fdxs1
Dnt: 1
Sec-Gpc: 1
Upgrade-Insecure-Requests: 1
X-Forwarded-For: XXX.XXX.XXX.XXX
X-Forwarded-Host: whoami.yourdomain
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: 7feee94dfc4d
X-Real-Ip: XXX.XXX.XXX.XXX
```

This indicate that the webservice was correctly published.

# Traefik: Applying TLS certificates with LetsEncrypt

* Now supose that you want to add a selfgenerated certificate, well, Traefik give you the option to configure a certresolver to autogenerate certificates with LetsEncrypt for your owned domains.

* To obtain this result you need to add this lines to the command section of the Traefik service. This configure the LetsEncrypt challenge to generate the certificates:

```
- "--certificatesresolvers.myresolver.acme.tlschallenge=true"
- "--certificatesresolvers.myresolver.acme.email=postmaster@example.com"
- "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
```

Remember to add your own email on the email section.

* To store the certificates you need to add also a volume or a path to store the files that Letsencrypt will generate.

```
volumes:
      - "./letsencrypt:/letsencrypt"
```

* Finally, to indicate that you want to add a particular endpoint or service TLS, you need to add these lines to your service label section. In this case, our whoami webservice.

```
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"
```

- We indicate the entrypoint for the https port (websecure on 443, defined on the Traefik section).
- We indicate also the certresolver that should generate a certificate for the defined domain.

* Our docker-compose.yml file should see like this:

```
version: "3.3"

services:

  traefik:
    image: "traefik:v2.4"
    container_name: "traefik"
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
	- "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=postmaster@example.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"
```

Now you can start your stack again with:

```
 docker-compose stop && docker-compose start -d
```

If the containers start well, you should see your whoami webservice secured by a LetsEncrypt TLS certificate.

# Traefik: Basic Auth for web services

* Next, we are going to add a basic web authentication to our webservice. To acomplish this we need to add some extra lines to our defined whoami simple webservice.

```
labels:
  - "traefik.http.routers.whoami.middlewares=auth
  - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/,test2:$$apr1$$d9hr9HBB$$4HxwgUir3HP4EsggP/QNo0"
```

These lines add the users option, that is an array for authorized users in the format name:encoded-password.

Passwords must be encoded using MD5, SHA1, or BCrypt.

You can use htpasswd to generate the passwords.

When the container was deployed with this conf, your http/https request needs to be authenticated and authorized to reach the web endopint for your webservice or application.

The entire file should looks like:

```
version: "3.3"

services:

  traefik:
    image: "traefik:v2.4"
    container_name: "traefik"
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
	- "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=postmaster@example.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"
	- "traefik.http.routers.whoami.middlewares=auth
      - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/,test2:$$apr1$$d9hr9HBB$$4HxwgUir3HP4EsggP/QNo0"
```

Now you can start your stack again with:

```
 docker-compose stop && docker-compose start -d
```

When the containers were started, access with your browser to your web endpoint configured on your domain and you should asked for introduce a user and pasword before accessing the webservice.

If you enter well the credentials that you configured, you should obtain the whoami webservice.

An option to authenticate to with basic_auth to the webservice endpoints, was using this format on your petitions or authentication configuration:

```
https://user:password@whoami.example.com
```

# Links

* TLS on Traefik with LetsEncrypt: https://doc.traefik.io/traefik/user-guides/docker-compose/acme-tls/
* Basic auth: https://doc.traefik.io/traefik/v2.0/middlewares/basicauth/