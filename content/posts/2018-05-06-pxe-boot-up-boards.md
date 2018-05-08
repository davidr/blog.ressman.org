---
date: 2018-05-05
lastmod: 2018-05-06
title: "PXE Booting x86_64 UEFI Clients with Dnsmasq"
description: "using UEFI PXE booting to install linux on up boards"
authors: ["davidr"]
categories:
  - tutorial
tags:
  - upboard
  - pxe
  - dnsmasq
toc: false
---

**Update:** *Naturally, the day after I get this up and running, Red Hat announces that [Atomic Host
will be supplanted by Container Linux](https://coreos.com/blog/coreos-tech-to-combine-with-red-hat-openshift),
so all of this stuff that's specific to Atomic is effectively deprecated. I'll be starting over with
Container Linux.*


I have a little [Intel NUC](https://www.intel.com/content/www/us/en/products/boards-kits/nuc.html) box I
use as a server. This is how I configured it to get the Up Boards to boot and install Fedora 
[Atomic](http://www.projectatomic.io/).

_TODO_: make these into ansible playbooks

# Firewalld Changes

For Dnsmasq to work, I need a zone for my k8s hosts and I need to open DNS, DHCP, and TFTP:

``` 
# DHCP is broadcast, so it isn't useful to add it to a zone. If I had multiple interfaces,
# I would restrict it to one
firewall-cmd --permanent --add-service=dhcp

firewall-cmd --permanent --new-zone=k8s
firewall-cmd --permanent --zone=k8s --add-source=10.0.0.0/24
firewall-cmd --permanent --zone=k8s --add-service=dns
firewall-cmd --permanent --zone=k8s --add-service=http
firewall-cmd --permanent --zone=k8s --add-service=syslog
firewall-cmd --permanent --zone=k8s --add-service=tftp
firewall-cmd --permanent --zone=k8s --add-port=68/udp

# pick up the config changes
firewall-cmd --reload
```

# Dnsmasq Setup

## TFTP

```
mkdir -p /var/lib/tftpboot
cp /boot/efi/EFI/fedora/{shim,grubx64}.efi /var/lib/tftpboot/
```

## Config

```
port=0

user=dnsmasq
group=dnsmasq

# DHCP pool setup
dhcp-range=10.0.0.151,10.0.0.200,12h
dhcp-option=option:router,10.0.0.1

# Static mappings
dhcp-host=cc:00:ff:ff:ee:ea,10.0.0.101,k8s-master01,infinite
dhcp-host=cc:00:ff:ff:ee:eb,10.0.0.102,k8s-master02,infinite

dhcp-host=cc:00:ff:ff:ee:ec,10.0.0.104,k8s-n01,infinite
dhcp-host=cc:00:ff:ff:ee:ed,10.0.0.105,k8s-n02,infinite
dhcp-host=cc:00:ff:ff:ee:ee,10.0.0.106,k8s-n03,infinite

dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-boot=tag:efi-x86_64,shim.efi

# TFTP Server setup
enable-tftp
tftp-root=/var/lib/tftpboot

log-dhcp
conf-dir=/etc/dnsmasq.d,.rpmnew,.rpmsave,.rpmorig
```

# Atomic Setup

## PXE Boot

### nginx

We need a web server to serve up the kickstart and install.img files:

```
dnf install -y nginx
mkdir -p /srv/http/{fedora,kickstart}
semanage fcontext -a -t httpd_sys_content_t "/srv/http(/.*)?"
restorecon -Rv /srv/http/

cat <<EOF > /etc/nginx/default.d/fedora.conf
location /fedora {
  alias /srv/http/fedora;
  autoindex on;
  autoindex_exact_size off;
}
EOF

cat <<EOF > /etc/nginx/default.d/kickstart.conf
location /kickstart {
  alias /srv/http/kickstart;
  autoindex on;
  autoindex_exact_size off;
}
EOF
```

### Install Files

#### kernel, initrd, install image, etc.
Now set up the files needed for kickstarting the boxes:

```
curl -OL https://download.fedoraproject.org/pub/alt/atomic/stable/Fedora-Atomic-28-20180425.0/AtomicHost/x86_64/iso/Fedora-AtomicHost-ostree-x86_64-28-20180425.0.iso

mount -t iso9660 -o loop Fedora-AtomicHost-ostree-x86_64-28-20180425.0.iso /mnt

mkdir -p /srv/http/fedora/28/atomic
rsync -a /mnt/images /srv/http/fedora/28/atomic/

mkdir -p /var/lib/tftpboot/fedora/28
cp /mnt/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/fedora/28/
```

#### grub 

```
cat <<EOF > /var/lib/tftpboot/grub.cfg
set default="0"

function load_video {
  insmod efi_gop
  insmod efi_uga
  insmod video_bochs
  insmod video_cirrus
  insmod all_video
}

load_video
set gfxpayload=keep
insmod gzio
insmod part_gpt
insmod ext2

set timeout=10
### END /etc/grub.d/00_header ###


### BEGIN /etc/grub.d/10_linux ###
menuentry 'Install Fedora 28 Atomic Host' --class fedora --class gnu-linux --class gnu --class os {
        linuxefi fedora/28/atomic/vmlinuz inst.stage2=http://10.0.0.10/fedora/28/atomic/ ip=dhcp inst.ks=http://10.0.0.10/kickstart/atomic-ks.cfg inst.cmdline inst.sshd
        initrdefi fedora/28/atomic/initrd.img
}
EOF
```

#### kickstart profiles

_TODO_: this needs to vary per-host to set up the static networking configuration correctly, but
the thought of installing something heavyweight like Cobbler or Foreman is... unappealing. I think
the right move is to just implement like a 20-line template rendering web server in golang or python.

```
cat <<EOF > /srv/http/kickstart/atomic-ks.cfg
text
install
reboot

sshpw --username root --plaintext installpw

auth --enableshadow --passalgo=sha512
ostreesetup --nogpg --osname="fedora-atomic" --remote="fedora-atomic" --url="file:///ostree/repo" --ref="fedora/28/x86_64/atomic-host"

ignoredisk --only-use=mmcblk0
keyboard us
lang en_US.UTF-8

network  --bootproto=static --device=enp2s0 --ip=10.0.0.104 --netmask=255.255.255.0 --hostname=k8s-n01.ressman.org --gateway=10.0.0.1 --nameserver 10.0.0.1

logging --host 10.0.0.10 --level debug

rootpw --iscrypted PASSWORD_CRYPT

timezone America/Chicago --isUtc

user --groups=wheel --name=atomic --password=PASSWORD_CRYPT --iscrypted --gecos="Atomic User"

sshkey --username root "SSH_KEY_1"
sshkey --username atomic "SSH_KEY_2"

bootloader --location=mbr --boot-drive=mmcblk0
clearpart --all --initlabel --drives=mmcblk0
autopart --noswap --fstype="xfs"

%post --erroronfail
rm -f /etc/ostree/remotes.d/fedora-atomic.conf
ostree remote add --set=gpgkeypath=/etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-28-primary fedora-atomic https://kojipkgs.fedoraproject.org/atomic/repo
cp /etc/skel/.bash* /root
%end
EOF
```


## OSTree

This was a little more complicated since I'm still learning about Atomic, but the goal here is
to have a periodically updating local copy of the ostree repo for all nodes to update against.
