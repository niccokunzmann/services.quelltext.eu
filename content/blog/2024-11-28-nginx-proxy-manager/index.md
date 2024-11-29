---
title: One Server Many Services - Nginx Proxy Manager
subtitle: "Nginx Proxy Manager allows hosting several services on one web server."
summary: Nginx Proxy Manager allows hosting several services on one web server. This blog post describes my setup.
date: 2024-11-28
cardimage: nginx-proxy-manager-card.png
featureimage: nginx-proxy-manager-feature.png
caption: Nginx Proxy Manager
authors:
  - Nicco: author.jpeg
---

[Nginx Proxy Manager] runs in front of all the services on my machine.
The image below shows a set of services that are running on the same machine.

{{< figArray subfolder="overview" figCaption="Nginx Proxy Mangaer allows managing many domains and redirect services." >}}

## Getting a Server

Since I am hosting the services with [Hetzner], I will have to add a domain configuration for the domain **quelltext.eu** that I bought.

{{< figArray subfolder="hetzner" figCaption="The Hetzner server has an IPv4 and an IPv6 address." >}}

## Redirecting a Domain

I use [selfhost.eu] or [thenames.co.uk] to buy my domains. There are surely more services and they all work the same.

The table below shows the IP address configuration for the place where I bought the domain.

| Domain | Record Type | Value |
| ------ | ----------- | ----- |
| *.hosted.quelltext.eu | A | 78.47.87.181 | 
| *.hosted.quelltext.eu | AAAA | 2a01:4f8:c012:e355::1 | 

After this is setup, it can take an hour for the information to propagate.
Then, all domains like these redirect to the one server.

- https://hosted.quelltext.eu - the domain itself
- https://open-web-calendar.hosted.quelltext.eu/ - a subdomain
- https://tor.open-web-calendar.hosted.quelltext.eu/ - a sub-subdomain

## Setting up the proxy manager

I installed [Docker](https://docs.docker.com/engine/install/) and [Docker Compose](https://docs.docker.com/compose/install/) on the server. I also recommend:

- git to version control your service changes
- [mosh](https://mosh.org/) to login on mobile
- disable ssh password access and use public key authentication instead

Then, I created this docker-compose file.

```sh
mkdir services
cd services
git init
nano docker-compose.yml
```

This

```yaml
# docker-compose.yml
services:
  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81' # This is the initial port for admin setup. Remove later.
      - '443:443'
    volumes:
      - ./nginx-proxy-manager/data:/data
      - ./nginx-proxy-manager/letsencrypt:/etc/letsencrypt
    networks:
      - default
```

Now, you can start the service with this command:

```sh
nicco@ubuntu-4gb-fsn1-1:~/services$ docker compose up -d
[+] Running 1/0
 ✔ Container services-nginx-proxy-manager-1    Running      0.0s 
```

## Firewall Configuration

Usually, a firewall blocks access to what is going on.
Ports 80 and 443 must be let through to the service.
The image below shows the firewall config.

{{< figArray subfolder="firewall" figCaption="Ports 443 and 80 must be configured in the firewall." >}}

After the firewall config, your service should be running under your domain.
In my case, I can visit [hosted.quelltext.eu] and see this:

{{< figArray subfolder="success" figCaption="Nginx Proxy Manager is running successfully." >}}


Now, you can use ssh port forwarding to configure the user account at port 81.


## Automatic Updates

I also have an **update.sh** file running in [cron job](https://phoenixnap.com/kb/set-up-cron-job-linux) in the same directory as the **docker-compose.yml**.

```sh
#!/bin/bash
#
# update the services
#

cd "`dirname \"$0\"`"

docker compose pull
docker compose create
docker compose up -d --remove-orphans
  
# clean up
# see https://stackoverflow.com/a/46159681/1320237
docker system prune -a -f
docker rm -v $(docker ps -a -q -f status=exited)
docker rmi -f  $(docker images -f "dangling=true" -q)
docker volume ls -qf dangling=true | xargs -r docker volume rm
```


## More services

By now, we can add a simple service, [Uptime Kuma]. That is the next blog post.


[Hetzner]: https://hetzner.cloud
[Nginx Proxy Manager]: https://nginxproxymanager.com/
[thenames.co.uk]: https://www.thenames.co.uk/
[selfhost.eu]: https://selfhost.eu/
[Uptime Kuma]: ../2024-11-29-uptime-kuma
[hosted.quelltext.eu]: https://hosted.quelltext.eu
