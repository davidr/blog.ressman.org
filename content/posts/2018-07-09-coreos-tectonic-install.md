---
date: 2018-07-09
lastmod: 2018-07-09
title: "Installing Tectonic/CoreOS on My Dev Cluster"
description: "getting linux and kubernetes bootstrapped on a cluster"
authors: ["davidr"]
categories:
  - tutorial
tags:
  - k8s
  - coreos
  - containerlinux
  - tectonic
toc: true
draft: true
---

Although it's being deprecated as it's merged into OpenShift, the Tectonic installer is still
pretty nice, and they've generously made it free for clusters of ten nodes or fewer. Since my little
dev cluster is six nodes, why not!?

I haven't used Terraform yet, so I'm just doing the regular 
[bare metal installation](https://coreos.com/tectonic/docs/latest/install/bare-metal/).

# Motivation

Since this is a dev cluster, I'll be reinstalling it, breaking it, putting OpenShift on it, moving it
back to CoreOS, etc. all the time so I'm writing this so I don't have to remember how to put CoreOS back
on it with Tectonic. 

# Prep

## Matchbox

Since I've [already got matchbox installed](/posts/2018-05-08-pxe-installing-container-linux/), this is pretty
much done. (Same with [PXE Boot](/posts/2018-05-06-pxe-boot-up-boards/).)

## DNS Records

I put records in my internal DNS server not just for the hosts themselves, but for the service names:

```
k8s-api IN CNAME k8s-master01.ressman.org.
k8s     IN CNAME k8s-n01.ressman.org.
```

## Downloading/Unpacking Tectonic Installer

```
[root@oliver tmp]# curl -sOL https://releases.tectonic.com/releases/tectonic_1.9.6-tectonic.1.zip
[root@oliver tmp]# unzip -q tectonic_1.9.6-tectonic.1.zip
[root@oliver tmp]# cd tectonic_1.9.6-tectonic.1/tectonic-installer
```

Then because `/dev/sda` is hardcoded into the installer but the Up Boards have `/dev/mmcrblk0`:

```
find . -name \*.tmpl -exec sed -e 's,/dev/sda,/dev/mmcrblk0,' -i {} \;
```

# Installing


Then just run the installer!

```
[root@oliver tectonic-installer]# linux/installer \
  -log-level debug \
  -address 0.0.0.0:4444 \
  -platforms bare-metal \
  -open-browser=false
```

