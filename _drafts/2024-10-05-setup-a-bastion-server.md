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


## Cluster provisioning

This bastion host now allows us to launch an initial fleet of servers that are not directly accessable from the internet, significantly reducing the attack surface of our cluster.

To accomplish this task, we need to define the proxy server configuration for the nodes. We'll do this by making two additions to our cloud-init.yaml.

```
runcmd:
  - echo "http_proxy=http://10.7.222.11:8888/" >> /etc/environment
  - echo "https_proxy=http://10.7.222.11:8888/" >> /etc/environment
  - echo "HTTP_PROXY=http://10.7.222.11:8888/" >> /etc/environment
  - echo "HTTPS_PROXY=http://10.7.222.11:8888/" >> /etc/environment
apt:
  http_proxy: http://10.7.222.11:8888/
  https_proxy: http://10.7.222.11:8888/
```

These should be included prior to any cloud-init stanza's that invoke external assets/packages.


Now, when we spin up our hosts, we'll be able to configure ssh to reach the hosts through the bastion server.

How many nodes do we spin up? This series of blogs will walk through setting up a cluster with stacked control plane nodes (vs using an external etcd cluster), which means we can select an odd number of control nodes.  One control node is obviously necessary for the cluster to start. For high availability, we would want 3 control nodes, so that the cluster will continue to operate smoothly even if (intentionally or otherwise) we have one control plane node down.

```
Host xd7imaster00
    HostName 10.7.222.13
    User ubuntu
    IdentityFile ~/.ssh/id_rsa_blog_skillcadet
    ProxyJump xd7tower

Host xd7inode00
    HostName 10.7.222.12
    User ubuntu
    IdentityFile ~/.ssh/id_rsa_blog_skillcadet
    ProxyJump xd7tower

Host xd7inode01
    HostName 10.7.222.14
    User ubuntu
    IdentityFile ~/.ssh/id_rsa_blog_skillcadet
    ProxyJump xd7tower
```

Finally, we add our internal hosts to the ssh config.  In addition to the Hostname information, we include the `ProxyJump` line so the ssh knows we reach the internal hosts via sshing through the tower host.

```
$ ssh xd7imaster00
The authenticity of host '10.7.222.13 (<no hostip for proxy command>)' can't be established.
ECDSA key fingerprint is SHA256:6lNvogJRmjYR9n1ZJs6KrCAXnGvsDmbW6RO0SG9DPvE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.7.222.13' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-31-generic x86_64)

ubuntu@imaster00:~$
```

Now, we should be able to ssh directly to the internal hosts.
