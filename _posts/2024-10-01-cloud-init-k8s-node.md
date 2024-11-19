---
layout: post
title: Bootstrapping Kubernetes nodes with cloud-init
---

## Freshness
Freshness - this, and all, k8s tutorial suffer from staleness quite quickly. In general, tutorials have been succesfully applied across versions, however this changes slightly with the new (2024) release structure of community supported kubernetes, where each version has it's own distribution url.  The GPG keys, SSH keys, and apt sources that will change will be highlighted in the document.

This version, dated October 1st, 2024, uses:

| Component  | Version    |
|------------|------------|
| Ubuntu     | v24.04 LTS |
| Kubernetes | v1.30      |
| Calico     | v3.28      |

## Why this post
There are plenty of tutorials out there that walk through manually setting up a kubernetes cluster, however, in the real world, we want to make things repeatable and not tedius. There are a few ways to provide this capability, most of them, like Anisible/Puppet/Chef/Salt/Terraform/Cloudformation all require additional infrastructure.  

But, what if there was a way to provide repeatable configuration, across cloud and virtualization platform providers, using a text file that can be checked into a code repository?

## cloud-init
Let me introduce you to [Cloud-Init](https://cloud-init.io/).  Using cloud-init, we will perform all the basic installation steps to bring a stock system image and prepare it to receive a kubernetes node.

We'll still have plenty to do to bring up and manage the cluster, however, we can start at the fun part and focus on what makes the nodes different.

Let's take a look at the [cloud-init.yaml](https://gist.github.com/ambled/f4c6949955724c6cb0f3e4fee164d086) that we'll be crafting.

### preamble, disable swap and (optional) customizable hostname
```
#cloud-config
hostname: inode01
create_hostname_file: true
mounts:
- [ swap ]
```

We start with the `#cloud-config` stanza.  Next we have an choice. We can create separate cloud-config.yaml files for each node and provide our own hostname, or, depending on your provider, we leave the hostname/create_hostname_file stanzas out and your node will be called something random or will be called `ubuntu` by default.  The hostname stanza is the only custom resource and if your hosts are called `ubuntu`, it's simple to change manually when you login to the host the first time.

Finally, we make sure that swap is disabled by providing no options in the config. Later, we'll run a command to make sure swap is currently off.
### Add admin user
```
users:
- name: ubuntu
  gecos: Ubuntu User
  shell: /bin/bash
  groups: users, admin
  sudo: ALL=(ALL) NOPASSWD:ALL
  ssh_authorized_keys:
    - ssh-rsa AAAAB3Nza......= troy@skillcadet
```

Next, we want to address the admin user. We want to disable root ssh login, so we need a regular user with admin privileges. We also want to secure our communication channel with a public/private key instead of using passwords for login.

```
  hashed_passwd: "$6$rounds=4096$/NM2KaSQ595tb74s$38kIiWZXh4MMpb95/0TdGAm.Wr7J9SMFnb.CSsxlsU0oE7pgkGcCs2JpJHDCuGXMNvVEmpU01SBONYmVxpAv8/"
  lock_passwd: false
  sudo: ALL=(ALL:ALL) ALL
```

You can, alternatively, replace the `sudo` stanza with the above triplet to enable password protected sudo. This isn't really ideal either, as providing a hash, even as strong as this, is very crackable if discovered, using a longer complex password could help. You can generate a hashed password with the following:

> mkpasswd -m sha-512 -R 4096

### Create system files
```
write_files:
- path: /etc/sysctl.d/kubernetes.conf
  owner: root:root
  permissions: '644'
  content: |
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
- path: /etc/modules-load.d/k8s.conf
  owner: root:root
  permissions: '644'
  content: |
    overlay
    br_netfilter
```

Additionally, we need a few configuration files to be written, luckly that is easily accomplished with cloud-init.
### update config changes


In the home stretch, we encounter the parts of the config that change.

| Resource                                   | Location                                                |
|--------------------------------------------|---------------------------------------------------------|
| The GPG for the containerd repository      | https://download.docker.com/linux/ubuntu/gpg            |
| The GPG for the kubernetes 1.30 repository | https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key |
| The repository for kubernetest 1.30        | https://pkgs.k8s.io/core:/stable:/v1.30/deb/            |

We'll add these into the correct locations below:

### Add apt sources
```
apt:
  sources:
    containerd.list:
      source: deb https://download.docker.com/linux/ubuntu $RELEASE stable
      key: |
        -----BEGIN PGP PUBLIC KEY BLOCK-----

        mQINBFit2ioBEADhWpZ8/wvZ6hUTiXOwQHXMAlaFHcPH9hAtr4F1y2+OYdbtMuth
        lqqwp028AqyY+PRfVMtSYMbjuQuu5byyKR01BbqYhuS3jtqQmljZ/bJvXqnmiVXh
            SNIPPED
        =0YYh
        -----END PGP PUBLIC KEY BLOCK-----
    k8s.list:
      source: deb https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
      key: |
        -----BEGIN PGP PUBLIC KEY BLOCK-----
        Version: GnuPG v2.0.15 (GNU/Linux)

        mQENBGMHoXcBCADukGOEQyleViOgtkMVa7hKifP6POCTh+98xNW4TfHK/nBJN2sm
        u4XaiUmtB9UuGt9jl8VxQg4hOMRf40coIwHsNwtSrc2R9v5Kgpvcv537QVIigVHH
             SNIPPED
        oyA0MELL0JQzEinixqxpZ1taOmVR/8pQVrqstqwqsp3RABaeZ80JbigUC29zJUVf
        =F4EX
        -----END PGP PUBLIC KEY BLOCK-----
```

Here, we introduce the two apt sources so we can retrieve the necessary packages.

> There are alternate ways to [specify apt sources](https://blog.skillcadet.com/extras/cloud-init-by-keyserver.html) instead of including the PGP keys directly.

### Add required packages
```
packages:
  - curl
  - gnupg2
  - software-properties-common
  - apt-transport-https
  - ca-certificates
  - containerd.io
  - kubeadm
  - kubelet
  - kubectl
  - fail2ban
```

### Fail2ban
All the packages above are consistent with most tutorials, with the expection of [fail2ban](https://github.com/fail2ban/fail2ban). This is added since we need to have an SSH port open to manage our system.  

Now, we've already disabled root login and disabled password logins in favor of SSH keypairs, however to make things even more secure, Fail2ban will monitor logs for ssh login failures (and other things if we canfigure fail2ban do so) and will block access to the SSH port for repeated offenders. The block goes away after a few minutes, so if you mess up a bunch of times in a row, just be patient. This wait period can frustrate a user, but it makes it impossible to brute force a login.

### Configure containerd, lock packages and start fail2ban
```
runcmd:
  - [ swapoff, -a ]
  - [ modprobe, overlay ]
  - [ modprobe, br_netfilter ]
  - [ sysctl, --system ]
  - containerd config default | tee /etc/containerd/config.toml >/dev/null 2>&1
  - [ sed, -i, 's/SystemdCgroup \= false/SystemdCgroup \= true/g', /etc/containerd/config.toml ]
  - [ systemctl, restart, containerd ]
  - [ apt-mark, hold, kubelet, kubeadm, kubectl ]
  - [ systemctl, enable, fail2ban ]
  - [ systemctl, start, fail2ban ]
```

Almost finished now, we execute some commands to prepare the system.

First, we apply system changes we made with write_files above. Then, we need to generate a containerd config file so we can override the default SystemdCgroup setting with 'sed'.
Next, we restart containerd to use the proper Cgroup.  We follow that with marking the kubernetes packages as HOLD so they do not get upgraded automatically. And then we enable and start fail2ban.

### Restart to start with a clean environment
```
power_state:
  mode: reboot
  message: First Boot Complete
```

Finally, we reboot to clear things up and start with a fresh boot, ready for us to login and create the cluster.

Coming soon, my tutorials on manually bringing up a cluster from this point and on connecting an external load balancer to a cluster nodeport.

---
[Github Issues](https://github.com/skillcadet/blog/issues) - available for suggestions/corrections
