---
layout: post
title: Verify SSH Fingerprint
---

## Infrastructure

In some deployments, we are able to establish a secure console to our nodes. This gives us the opportunity to retrieve (and perhaps publish) our SSH fingerprints so that we can offer trust in adding SSH keys from new hosts.

### Scan new node

```
xd7tower:~$  ssh inode00
The authenticity of host 'inode00 (10.7.200.13)' can't be established.
ED25519 key fingerprint is SHA256:aOvD6mO371xxyFbFlkm49z+G5kdj8sdpts7SX77g+XA.
```

We begin by retrieving the fingerprints using `ssh`.

Reviewing the template for the fingerprints we have:
> `algorithm` key fingerprint is  [`hashtype`]:`hash`

Where the fingerprint hash can come in two flavors:
* MD5 `XX:XX:XX:..:XX`  or `XXXXXX..XX`
* SHA256 `[A-Z][a-z][0-9][+/=]`

## On the node

Now that we have the required information, we can retrieve the servers fingerprints.

Each fingerprint can be evaluated from the system generated public keys located in /etc/ssh.

The path format for each public key is:

> /etc/ssh/ssh_host_`algorithm`_key.pub

### MD5

Although not commonly used, some older hosts still generate MD5 fingerprints

> ssh-keygen -E md5 -lf `public_key_filename`

### SHA256

> ssh-keygen -lf `public_key_filename`

### Example

```
ubuntu@inode00:/etc/ssh$ ssh-keygen -E md5 -lf /etc/ssh/ssh_host_ed25519_key.pub
256 MD5:0c:0b:a9:88:81:9b:5d:e2:25:04:f9:e9:ce:c2:48:37 root@ubuntu (ED25519)
ubuntu@inode00:/etc/ssh$ ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:aOvD6mO371xxyFbFlkm49z+G5kdj8sdpts7SX77g+XA root@ubuntu (ED25519)
```

### Visual ASCII art

```
ubuntu@inode00:~$ ssh-keygen -t MD5 -vlf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:aOvD6mO371xxyFbFlkm49z+G5kdj8sdpts7SX77g+XA root@ubuntu (ED25519)
+--[ED25519 256]--+
|             +oo |
|            ..=  |
|            .o   |
|       . . o. .  |
|      o S = .. . |
|     . . . o . +.|
|     ..   .  oBE=|
|    o.+. .  .+*@=|
|   oo+o=+   o+O*O|
+----[SHA256]-----+
ubuntu@inode00:~$ ssh-keygen -E MD5 -vlf /etc/ssh/ssh_host_ed25519_key.pub
256 MD5:0c:0b:a9:88:81:9b:5d:e2:25:04:f9:e9:ce:c2:48:37 root@ubuntu (ED25519)
+--[ED25519 256]--+
|.o.              |
|o.   .           |
|o.o.= .          |
|.Bo* . +         |
|=.+   . S        |
| ..E             |
|+o. .            |
|o.o              |
| .               |
+------[MD5]------+
```

Adding a `v` to our flags will generate the ASCII art that represents the public key.
