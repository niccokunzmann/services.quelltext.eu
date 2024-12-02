---
title: Setup your own Unified Push Service for Telegram
subtitle: "I setup Unified Push service ntfy in a docker container behind Nginx Proxy Manager."
summary: "I setup Unified Push service ntfy in a docker container behind Nginx Proxy Manager."
date: 2024-12-02
cardimage: ntfy-card.png
featureimage: ntfy-feature.png
caption: The NTFY service running.
authors:
  - Nicco: author.jpeg
---

Telegram works without Unified Push. However, this means that [Google, Apple and law enforcement can track your location and your IP address using Apple's or Google's Push Notifications][push-id]. This is a problem for privacy.
You can opt out from this surveillance by using [Unified Push] and hosting a Unifies Push Service yourself.
This blog post describes how to host it and configure Telegram, Signal, Matrix and Mastodon to use it.

## Setup NTFY as a Docker Container


In the **docker-compose.yml** that I created in [first blog post][first], I add this:

```yaml
# docker-compose.yml
services:
  # ...
  ntfy:
    image: binwiederhier/ntfy
    container_name: ntfy
    command:
      - serve
    environment:
      - TZ=Europe/London    # optional: set desired timezone
#    user: UID:GID # optional: replace with your own user/group or uid/gid
    volumes:
      - ./ntfy/cache:/var/cache/ntfy
      - ./ntfy/etc:/etc/ntfy
#    ports:
#      - 80:80
    healthcheck: # optional: remember to adapt the host:port to your environment
        test: ["CMD-SHELL", "wget -q --tries=1 http://localhost:80/v1/health -O - | grep -Eo '\"healthy\"\\s*:\\s*true' || exit 1"]
        interval: 60s
        timeout: 10s
        retries: 3
        start_period: 40s
    restart: unless-stopped

```

This stores all the data of this push service in the **./ntfy** directory.
This starts the service:

```sh
services$ docker compose up -d
[+] Running 2/0
 ✔ Container services-ntfy-1                   Running      0.0s 
 ✔ Container services-nginx-proxy-manager-1    Running      0.0s 
```

## Configure Nginx Proxy Manager for NTFY

{{< figArray subfolder="npm" figCaption="Configure NGINX Proxy Manager to expose ntfy." >}}

We had configured that all sub-domains of hosted.quelltext.eu get directed to the server.
In the screenshots above you can see that we add websocket support and request a new certificate for
[ntfy.hosted.quelltext.eu].
The name to redirect to is again the name of the docker container since they are in the same docker network **services-default**.

When this service is successfully running, you should see it on your configured domain. We will continue to use
[ntfy.hosted.quelltext.eu]. You can do so, too.

## Install the NTFY App

You should install the [NTFY app](https://github.com/binwiederhier/ntfy-android#readme) to use the self-hosted service.

Go to Settings ➡ General and set these values:

- Default Server: https://ntfy.hosted.quelltext.eu/

This should allow you now, to register with the server.

## Configure Telegram to use Unified Push

Sadly, the main Telegram App does not support Unified Push.
However, the fork Mercurygram does support it. Here are some links:

- [Install Mercurygram from F-Droid](https://f-droid.org/packages/it.belloworld.mercurygram/)
- [Mercurygram Source Code Repository](https://github.com/Mercurygram/Mercurygram)
- [Mercurygram on Appteka](https://appteka.store/app/077r155091)

Mercurygram works like Telegram. In Settings ➡ Notifications and Sounds ➡ Other, you can configure to use Unified Push.
At the point of writing this, [I could not configure my own NTFY server](https://github.com/Mercurygram/Mercurygram/issues/55).

## Configure Signal to use Unified Push

We need an adapter "Mollydocket" for this, if we want to self-host this.
Again, add this to the **docker-compose.yml** and run **docker compose up -d**:

```yaml
  mollysocket:
    # from https://github.com/mollyim/mollysocket/blob/main/docker-compose.yml
    image: ghcr.io/mollyim/mollysocket:1
    container_name: mollysocket
    restart: always
    volumes:
      - ./mollysocket:/data
    working_dir: /data
    command: server
    environment:
      - MOLLY_DB="/data/mollysocket.db"
      # Do not add space in the array ["http://a.tld","*"]
      - MOLLY_ALLOWED_ENDPOINTS=["*"]
      - MOLLY_ALLOWED_UUIDS=["*"]
      - MOLLY_HOST=0.0.0.0
      - MOLLY_PORT=8020
      - RUST_LOG=info
#    ports:
#      - "8020:8020"
```

{{< figArray subfolder="molly" figCaption="Configure NGINX Proxy Manager to expose mollysocket." >}}

This service is then running under [mollysocket.hosted.quelltext.eu].

Again, by default, Signal does not support Unified Push. However, [Molly] does.
We can add the Molly Repository to our F-Droid app as described on their website.

Then, we can change to our own hosted mollysocket instance in Settings ➡ Notifications ➡ Push Strategy.

- Delivery Method: Unified Push
- Click on Unified Push ➡
  - Notification Method: ntfy
  - Server URL: https://mollysocket.hosted.quelltext.eu/

Now, the **Status** should show: OK

## Configure Matrix to use Unified Push

{{< figArray subfolder="matrix" figCaption="Schildichat is configured to use NTFY." >}}

I use [Schildichat](https://schildi.chat/) to chat on Matrix servers.
It has Unified Push built in.

Go to Settings ➡ Notifications ➡ Notifications configuration and choose **ntfy** as Notification method.

## Configure Mastodon to use Unified Push

In order to use Mastodon with Unified Push, you will need to use an client App that supports Unified Push.
I use [Pachli](https://f-droid.org/en/packages/app.pachli/) for this reason.

Go to Preferences ➡ Notifications and configure ntfy.

{{< figArray subfolder="pachli" figCaption="Mastodon client Pachli configured to use my NTFY service." >}}

Now, you can follow [@unifiedpush@fosstodon.org](https://fosstodon.org/@unifiedpush).

## Summary

In this blog post, we show how to setup your own Unified Push infrastructure and migrate the some of the most common apps to use it.

[push-id]: https://www.wired.com/story/apple-google-push-notification-surveillance/
[Unified Push]: https://unifiedpush.org/
[first]: ../2024-11-28-nginx-proxy-manager/
[ntfy.hosted.quelltext.eu]: https://ntfy.hosted.quelltext.eu/
[mollysocket.hosted.quelltext.eu]: https://mollysocket.hosted.quelltext.eu/
[Molly]: https://molly.im/