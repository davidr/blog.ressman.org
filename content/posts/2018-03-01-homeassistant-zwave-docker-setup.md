---
date: 2018-03-01
lastmod: 2018-03-01
title: "Setting up Home Assistant and Z-Wave on Docker"
description: "tutorial for setting up the pieces to make home assistant work correctly with z-wave inside docker containers"
authors: ["davidr"]
categories:
  - tutorial
tags:
  - homeassistant
  - docker
  - zwave
toc: false
---

# Docker host changes

While in general, I wanted anything configurable to be configured inside the Docker container, I did need
to make some small changes on the docker host itself.

## Z-Wave stick device name

So that I know what device to pass into the openzwave docker container, I want to have a consistent device
name for my [Aeotec Z-Stick Gen5](https://aeotec.com/z-wave-usb-stick), `/dev/z-stick`

When I plugged it in, my kernel log showed these messages:

```
[ 3015.445703] usb 1-2: new full-speed USB device number 5 using xhci_hcd
[ 3015.572982] usb 1-2: New USB device found, idVendor=0658, idProduct=0200
[ 3015.572991] usb 1-2: New USB device strings: Mfr=0, Product=0, SerialNumber=0
[ 3015.590106] cdc_acm 1-2:1.0: ttyACM0: USB ACM device
```

and `lsusb` shows this corresponding entry on device number 5 as referenced above:

```
Bus 001 Device 005: ID 0658:0200 Sigma Designs, Inc. 
```

with `0658` being the vendor id and `0200` being the product id of the Z-Stick. Creating `/etc/udev/rules.d/99-z-stick.rules`
with the following contents:

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="0658", ATTRS{idProduct}=="0200", SYMLINK+="z-stick"
```

followed by a `udevadm trigger` does the trick!

```
[root@oliver ~]# ls -l /dev/z-stick 
lrwxrwxrwx. 1 root root 7 Mar  1 14:19 /dev/z-stick -> ttyACM0
```



# openzwave container

I altered [Rui Marinho's openzwave docker image](https://github.com/ruimarinho/docker-openzwave)
a bit to shrink the image down, but then I realized it's no longer necessary to use openzwave, so
there went a couple hours right down the drain

# Home Assistant Container

This seems to work mostly out of the box with docker:

```
docker run \
  -d \
  --name=home-assistant \
  -v /srv/homeassistant:/config:z \
  -v /etc/localtime:/etc/localtime:ro \
  --device /dev/z-stick:/dev/z-stick \
  --net=host \
  homeassistant/home-assistant:latest
```

_Note_: that `:z` at the end of my configuration volume is needed so that Docker updates the SELinux
label on that directory. Otherwise, SELinux will refuse to let Docker mount it

Poke a hole in the firewall:

```
[root@oliver]/srv# firewall-cmd --permanent --add-port=8123/tcp
success
```
