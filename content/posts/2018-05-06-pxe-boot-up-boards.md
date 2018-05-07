---
date: 2018-05-05
lastmod: 2018-05-06
title: "PXE Booting x86_64 UEFI Clients with Dnsmasq"
description: "using UEFI PXE booting to install linux on up boards"
authors: ["davidr"]
draft: true
categories:
  - tutorial
tags:
  - upboard
  - pxe
  - dnsmasq
toc: false
---

To get an OS on the boxes, I need to PXE boot them and install Linux

# Firewalld Changes

For Dnsmasq to work, I need a zone for my k8s hosts and I need to open DNS, DHCP, and TFTP:

``` 
# DHCP is broadcast, so it isn't useful to add it to a zone. If I had multiple interfaces,
# I would restrict it to one
firewall-cmd --permanent --add-service=dhcp

firewall-cmd --permanent --new-zone=k8s
firewall-cmd --permanent --zone=k8s --add-source=192.168.200.0/24
firewall-cmd --permanent --zone=k8s --add-service=tftp
firewall-cmd --permanent --zone=k8s --add-service=dns
firewall-cmd --permanent --zone=k8s --add-port=68/udp

# pick up the config changes
firewall-cmd --reload
```

# Dnsmasq Setup

## TFTP

```
mkdir /var/lib/tftpboot/uefi
cp /boot/efi/EFI/fedora/{shim,grubx64}.efi /var/lib/tftpboot/uefi/
```

# Atomic Setup

## PXE Boot

```
curl -OL https://download.fedoraproject.org/pub/alt/atomic/stable/Fedora-Atomic-28-20180425.0/AtomicHost/x86_64/iso/Fedora-AtomicHost-ostree-x86_64-28-20180425.0.iso

mount -t iso9660 -o loop Fedora-AtomicHost-ostree-x86_64-28-20180425.0.iso /mnt

mkdir -p /srv/http/fedora/atomic/28/images/pxeboot
cp /mnt/images/pxeboot/{vmlinuz,initrd.img}  /srv/http/fedora/atomic/28/images/pxeboot/

mkdir -p /var/lib/tftpboot/fedora/28
cp /mnt/images/pxeboot/vmlinuz /var/lib/tftpboot/fedora/28/
```

## OSTree

