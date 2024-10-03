---
layout: post
title: Cloud-init by keyserver
---

# Cloud-init by keyserver

If you prefer to not import the PGP Blocks into your yaml, there is an alternative.

It is a bit more involved and also subject to the changes in URL and KeyID with each version, or, as K8s points out, "since it is community run, anything can change.".

## Runcmd
You could write your own `runcmd:` that attempts to perform the following operations:
```
runcmd:
 - curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmour -o /etc/apt/trusted.gpg.d/containerd.gpg
 - curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/k8s.gpg
```

However, cloud-init offers another way to describe the keys.

## Keyserver
```
apt:
  sources:
    containerd.list:
      source: deb https://download.docker.com/linux/ubuntu $RELEASE stable
      keyserver: https://download.docker.com/linux/ubuntu/gpg
      keyid: 8D81803C0EBFCD88
    k8s.list:
      source: deb https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
      keyserer: https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb/Release.key
      keyid: 234654DA9A296436
```

This is easier to read, though it requires two additional steps and the tool `gpg` to acquire the necessary information.
### Docker PGP keyid
```
$ curl -s -L https://download.docker.com/linux/ubuntu/gpg |gpg --verbose --dry-run --import
gpg: pub  rsa4096/8D81803C0EBFCD88 2017-02-22  Docker Release (CE deb) <docker@docker.com>
gpg: Total number processed: 1
```

To retrieve the containerd.list.keyid, see the result rsa4096/**D81803C0EBFCD88**
### Kubernetes PGP keyid
```
$ curl -s -L https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key |gpg --verbose --dry-run --import
gpg: armor header: Version: GnuPG v2.0.15 (GNU/Linux)
gpg: pub  rsa2048/234654DA9A296436 2022-08-25  isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
gpg: key 234654DA9A296436: 1 signature not checked due to a missing key
gpg: Total number processed: 1
```

To retrieve the k8s.list.keyid, we need to add -L to the curl, to follow the redirect to the CDN and then the result rsa2048/**234654DA9A296436**
### Kubernetes PGP CDN location
```
curl -s -D -  https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key |grep location
location: https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb/Release.key
```

gpg can't follow a redirect, so we have to give the current CDN location as the keyserver.

## Conclusion

There are alternative ways to specify the PGP keys, however they require additional steps, another tool AND they require the keyservers to be reachable, etc.

I prefer the declaritive way of including the keys, since they change frequently anyway, so a way to update them is necessary and it saves a couple external requests from each cloud-init process.
