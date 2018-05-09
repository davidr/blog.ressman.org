---
date: 2018-05-08
lastmod: 2018-05-08
title: "Using Matchbox to Install Container Linux"
description: "using matchbox to install linux on up boards"
authors: ["davidr"]
categories:
  - tutorial
tags:
  - upboard
  - pxe
  - dnsmasq
  - containerlinux
  - coreos
draft: true
toc: false
---

# Installation

## Matchbox

### Aborted Copr Attempt:

My pre-existing server is running Fedora, so to get matchbox installed, I tried to do it the easy way by 
enabling the Copr repo:

```
dnf copr enable @CoreOS/matchbox
dnf install matchbox
```

Naturally, this gives an error:

```
Failed to synchronize cache for repo 'group_CoreOS-matchbox', disabling.
```

It turns out the one in the Copr repo is a year old anyway, so it's no big loss.

### Via Docker:

Open up port 8080 and 8081:

```
firewall-cmd --permanent --zone=k8s --add-port=8080/tcp
firewall-cmd --permanent --zone=k8s --add-port=8081/tcp
```

Using my LetsEncrypt wildcart keys, I set up `/etc/matchbox`:

```
cp /etc/letsencrypt/archive/ressman.org/privkey.pem /etc/matchbox/server.key
cp /etc/letsencrypt/archive/ressman.org/fullchain1.pem /etc/matchbox/ca.crt
cp /etc/letsencrypt/archive/ressman.org/cert1.pem /etc/matchbox/server.crt
```

and up we go with the container:

```
# docker run \
    --net=host \
    --rm \
    -v /var/lib/matchbox:/var/lib/matchbox:Z \
    -v /etc/matchbox:/etc/matchbox:Z,ro \
    quay.io/coreos/matchbox:latest \
    -address=0.0.0.0:8080 \
    -rpc-address=0.0.0.0:8081 \
    -log-level=debug

time="2018-05-08T20:22:14Z" level=info msg="Starting matchbox gRPC server on 0.0.0.0:8081" 
time="2018-05-08T20:22:14Z" level=info msg="Using TLS server certificate: /etc/matchbox/server.crt" 
time="2018-05-08T20:22:14Z" level=info msg="Using TLS server key: /etc/matchbox/server.key" 
time="2018-05-08T20:22:14Z" level=info msg="Using CA certificate: /etc/matchbox/ca.crt to authenticate client certificates"
```

Looks good!

### Run Container Automatically

I just wanted to get it up and running, so instead of doing anything fancy with podman or
runc or anything, I just made a `/etc/systemd/system/docker.matchbox` systemd unit file:

```
[Unit]
Description=matchbox (Docker)
After=docker.service
Requires=docker.service
 
[Service]
TimeoutStartSec=0
Restart=always

ExecStartPre=-/usr/bin/docker kill matchbox
ExecStartPre=-/usr/bin/docker rm matchbox
ExecStartPre=-/usr/bin/docker pull "quay.io/coreos/matchbox:v0.7.0"
ExecStart=/usr/bin/docker run --name=matchbox --net=host --rm -v /var/lib/matchbox:/var/lib/matchbox:Z -v /etc/matchbox:/etc/matchbox:Z,ro quay.io/coreos/matchbox:latest -address=0.0.0.0:8080 -rpc-address=0.0.0.0:8081 -log-level=debug
ExecStop=/usr/bin/docker stop matchbox
 
[Install]
WantedBy=multi-user.target

```

Okay, so matchbox is installed and running (I assume) correctly. Let's give it something to do:

