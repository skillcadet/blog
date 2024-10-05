---
layout: post
title: Setup a bastion host
---

We'll begin the process of bringing up a private cluster by creating a host that exists on both the public network and an internal private network.

Create a new instance with an external ip address or that has external ports mapped for 22, 80 and 443.  Create an internal ip address on the same or a second interface.
Initialize the instance to include privoxy and nginx using [cloud-init](https://github.com/iweave/xd7k8s/blob/main/cloud-init/cloud-init-ubuntu2404-proxy-xd7tower.yaml).
Now we provision the host in the cluster so we can get our initial services online. (This also establishes the internal ip address if you don't choose it in advance)


### SSH config

We're going to be making changes to the local ~/.ssh/config file to facilite communicating to our cluster.

```
Host xd7tower
    HostName 162.200.24.101
    User ubuntu
    IdentityFile ~/.ssh/id_rsa_blog_skillcadet
```

Now we ssh to the tower machine for the first time:

```
ssh xd7tower
The authenticity of host '162.200.24.101 (162.200.24.101)' can't be established.
ECDSA key fingerprint is SHA256:k6AjX1v7wDxj/QYffwiHdAKS8rR2cGIMTIQEben5UH8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '162.200.24.101' (ECDSA) to the list of known hosts.
```

We'll need to approve the host fingerprint, which we often need to do blindly if we don't take the step of assigning the host keys in advance.

### Privoxy config

```
echo "listen-address  10.7.222.11:8888" |sudo tee -a /etc/privoxy/config
sudo service privoxy restart
```

Now we make one small change to privoxy. We will add a listener on the internal ip address. Then we restart privoxy to pick up the change.

Later, in another post, we'll use NGINX as an external load balancer to our private cluster.


