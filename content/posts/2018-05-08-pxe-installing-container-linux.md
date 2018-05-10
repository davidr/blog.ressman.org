---
date: 2018-05-10
lastmod: 2018-05-10
title: "Using Matchbox to Install Container Linux"
description: "netbooting a collection of Up boards and installing CoreOS Container Linux on them"
authors: ["davidr"]
categories:
  - tutorial
tags:
  - upboard
  - pxe
  - dnsmasq
  - containerlinux
  - coreos
  - k8s
toc: true
---

# Installation

## Matchbox

> [matchbox](https://coreos.com/matchbox/docs/latest/matchbox.html) is an
> HTTP and gRPC service that renders signed Ignition configs, cloud-configs,
> network boot configs, and metadata to machines to create CoreOS Container
> Linux clusters.

### Aborted Copr Attempt:

My pre-existing server is running Fedora, so to get matchbox installed,
I tried to do it the easy way by enabling the Copr repo:

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

### Configuration

Okay, so matchbox is installed and running (I assume) correctly. Let's give it something to do:


``` bash
mkdir -p /var/lib/matchbox/{ignition,generic,groups,profiles,assets}
```

#### Assets

Download some CoreOS images:

```
[root@oliver tmp]# curl -sOL https://raw.githubusercontent.com/coreos/matchbox/master/scripts/get-coreos
[root@oliver tmp]# bash get-coreos stable 1688.5.3 /var/lib/matchbox/assets
Creating directory /var/lib/matchbox/assets/coreos/1688.5.3
Downloading CoreOS stable 1688.5.3 images and sigs to /var/lib/matchbox/assets/coreos/1688.5.3
CoreOS Image Signing Key
####################################################################################################### 100.0%
gpg: key 93D2DCB4: "CoreOS Buildbot (Offical Builds) <buildbot@coreos.com>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1
version.txt
####################################################################################################### 100.0%
coreos_production_pxe.vmlinuz...
####################################################################################################### 100.0%
coreos_production_pxe.vmlinuz.sig
####################################################################################################### 100.0%
coreos_production_pxe_image.cpio.gz
####################################################################################################### 100.0%
coreos_production_pxe_image.cpio.gz.sig
####################################################################################################### 100.0%
coreos_production_image.bin.bz2
####################################################################################################### 100.0%
coreos_production_image.bin.bz2.sig
####################################################################################################### 100.0%
```

#### Groups

> Groups define selectors which match zero or more machines. Machine(s) matching a group will
> boot and provision according to the group's `Profile`

```
[root@oliver groups]# jq . /var/lib/matchbox/groups/k8s-master01.json
{
  "name": "k8s-master01",
  "profile": "etcd",
  "selector": {
    "mac": "00:07:32:4e:0c:67"
  },
  "metadata": {
    "domain_name": "k8s-master01.ressman.org",
    "fleet_metadata": "role=etcd,name=k8s-master01",
    "etcd_name": "k8s-master01",
    "etcd_initial_cluster": "node1=http://k8s-master01.ressman.org:2380"
  }
}
```

#### Profiles

> Profiles reference an Ignition config by name and define network boot settings

```
[root@oliver groups]# jq . /var/lib/matchbox/profiles/etcd.json
{
  "id": "etcd",
  "name": "Container Linux with etcd3",
  "ignition_id": "etcd3.yaml",
  "boot": {
    "kernel": "/assets/coreos/1688.5.3/coreos_production_pxe.vmlinuz",
    "initrd": [
      "/assets/coreos/1688.5.3/coreos_production_pxe_image.cpio.gz"
    ],
    "args": [
      "coreos.config.url=http://oliver.ressman.org:8080/ignition?uuid=${uuid}&mac=${mac:hexhyp}",
      "coreos.first_boot=yes",
      "coreos.autologin"
    ]
  }
}
```

#### Ignition/Container Linux Config Templates

Using the example template:

```
[root@oliver matchbox]# cat /var/lib/matchbox/ignition/etcd3.yaml 
---
systemd:
  units:
    - name: etcd-member.service
      enable: true
      dropins:
        - name: 40-etcd-cluster.conf
          contents: |
            [Service]
            Environment="ETCD_IMAGE_TAG=v3.2.0"
            Environment="ETCD_NAME={{.etcd_name}}"
            Environment="ETCD_ADVERTISE_CLIENT_URLS=http://{{.domain_name}}:2379"
            Environment="ETCD_INITIAL_ADVERTISE_PEER_URLS=http://{{.domain_name}}:2380"
            Environment="ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379"
            Environment="ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380"
            Environment="ETCD_INITIAL_CLUSTER={{.etcd_initial_cluster}}"
            Environment="ETCD_STRICT_RECONFIG_CHECK=true"
    - name: locksmithd.service
      dropins:
        - name: 40-etcd-lock.conf
          contents: |
            [Service]
            Environment="REBOOT_STRATEGY=etcd-lock"
{{ if index . "ssh_authorized_keys" }}
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        {{ range $element := .ssh_authorized_keys }}
        - {{$element}}
        {{end}}
{{end}}
```

### Testing

A test request seems to indicate it's working okay:

```
[root@oliver matchbox]# curl -s 'http://oliver.ressman.org:8080/ignition?mac=00:07:32:4e:0c:67' | jq .
{
  "ignition": {
    "config": {},
    "timeouts": {},
    "version": "2.1.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {
    "units": [
      {
        "dropins": [
          {
            "contents": "[Service]\nEnvironment=\"ETCD_IMAGE_TAG=v3.2.0\"\nEnvironment=\"ETCD_NAME=k8s-master01\"\nEnvironment=\"ETCD_ADVERTISE_CLIENT_URLS=http://k8s-master01.ressman.org:2379\"\nEnvironment=\"ETCD_INITIAL_ADVERTISE_PEER_URLS=http://k8s-master01.ressman.org:2380\"\nEnvironment=\"ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379\"\nEnvironment=\"ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380\"\nEnvironment=\"ETCD_INITIAL_CLUSTER=node1=http://k8s-master01.ressman.org:2380\"\nEnvironment=\"ETCD_STRICT_RECONFIG_CHECK=true\"\n",
            "name": "40-etcd-cluster.conf"
          }
        ],
        "enable": true,
        "name": "etcd-member.service"
      },
      {
        "dropins": [
          {
            "contents": "[Service]\nEnvironment=\"REBOOT_STRATEGY=etcd-lock\"\n",
            "name": "40-etcd-lock.conf"
          }
        ],
        "name": "locksmithd.service"
      }
    ]
  }
}
```

This is new to me, so I don't know if this is right, but at least it parses, so that must
be a good sign, right?

## Inevitable Failure

Actually, everything works pretty well all-in-all. The boxes boot, PXE, download iPXE, download
the correct ignition configs and get the kernel and initrd.

They're currently hanging at boot with this error:

```
dev-disk-by\x2dlabel-OEM.device: Job dev-disk-by\x2dlabel-OEM.device/start timed out.
```

But that's in Linux, so I'm reasonably please about the whole thing. I'll troubleshoot this
error for a while but probably re-redeploy the cluster on Fedora Atomic just so I can get
working. Fortunately, when I fix this, it will be trivial to go back and forth between Atomic
and Container Linux.
