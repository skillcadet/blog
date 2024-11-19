---
layout: post
title: Starting an on-premise kubernetes cluster
---

In this series of posts, we are building an on-premise kubernetes cluster. This is a manual configuartion as we presume no support for BGP in virtualized or non-enterprise networking. In the [first](https://blog.skillcadet.com/2024/10/01/cloud-init-k8s-node.html) and [second](/edge.html) posts we seeded our initial servers using the cloud-init framework.

Depending on your own infrastructure needs, you may need to provision the cluster before you get the IP Addresses necessary for /etc/hosts.

This series uses a public network and an internal private network, as my internal network connections are 100GB and unmetered, which gives our inter-node communications a boost. For /etc/hosts, I used the internal ip addresses.  In practice, my nodes still need an external ip address to request packages/DNS/etc, but I have firewalled all inbound connections on the public networks and access the internal ip addresses using a bastion host that sits on both networks and lets ports 22/80/443 in. When I do this in production, my ssh interface is one of two nodes dedicated to cluster access and the 80/443 reverse proxies are seperate nodes without public ssh access.

In this post, I will take one other shortcut. I'm using the master nodes ip address for the control plane endpoint. In production, I would setup another, internal load balancer pointing to my multiple control plane nodes. This tutorial DOES setup the cluster to be upgradeable to high availability by adding more nodes (something to be covered in a future post).

### Infrastructure
| ip address | hostnames               |
|------------|-------------------------|
| 10.0.0.11  | xd7tower                |
| 10.0.0.12  | xd7imaster00 xd7control |
| 10.0.0.13  | xd7inode00              |
| 10.0.0.14  | xd7inode01              |


## Initializing the cluster

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint "xd7control:6443" --upload-certs --apiserver-advertise-address 10.7.222.12
```

Let's break this down.  

We're going to be using Calico which defaults to a pod network of 192.168.0.0/16, so we create the cluster that way.

Control-plane-endpoint is something we can't add to the cluster after it's created, so we define it now using our /etc/hosts entry to make things work.
Upload-certs is helpful if we're going to be adding more control plane nodes in the next few hours (We can always restart the join process to add nodes after this window has expired).
Apiserver-advertise-address is set to the internal address of the node since we want to use the internal network and kubernetes will by default pick an address on the network interface with the default route.

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

Next, we'll follow the instructions to clone the kubeconfig information to our local user. Through various means for port forwarding that will be addressed in another post, you can also clone the kubeconfig to your local machine to have remote access to `kubectl`.

```
ubuntu@imaster00:~$ kubectl get nodes -o wide
NAME        STATUS     ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION     CONTAINER-RUNTIME
imaster00   NotReady   control-plane   5m    v1.30.5   10.0.0.12   <none>        Ubuntu 24.04 LTS   6.8.0-31-generic   containerd://1.7.22
```

And we test `kubectl`

### Calico

Kubernetes needs a Container Network Interface as a layer of abstraction. While it's common to see Calico as the CNI, most tutorials specify just the raw manifests to get a cluster up and running instead of using the full control operator, this is mentioned in the Calico website:

> Calico can also be installed using raw manifests as an alternative to the operator. The manifests contain the necessary resources for installing Calico on each node in your Kubernetes cluster. Using manifests is not recommended as they cannot automatically manage the lifecycle of the Calico as the operator does. However, manifests may be useful for clusters that require highly specific modifications to the underlying Kubernetes resources.

Since we don't have any specific needs other than wanting to use the internal network, and we want a cluster we can upgrade, we'll install the operator and then request the network to be provisioned.

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml 
```

```
kubectl get pods -A -w
```

We can monitor the network coming up as we see first the `calico-system` and then the `calico-apiserver` pods go through the phases until they reach `Running`. Hit `Control+C` to exit the watch.

```
ubuntu@imaster00:~$ kubectl get nodes
NAME        STATUS   ROLES           AGE   VERSION
imaster00   Ready    control-plane   64m   v1.30.5
```

Now that networking is fully functional, we can see the initial node is in a ready state.

### More control plane nodes

Now that we have configured networking, we can begin adding more nodes. We would start by adding more control plane nodes by using the `kubeadm join` command that adds control-plane and certificate-key parameters. We're not going to be adding additional control plane nodes in the tutorial.

## Worker Nodes

```
sudo kubeadm join xd7control:6443 --token ui1vd6.kvea6irgh5ulrka1 \
	--discovery-token-ca-cert-hash sha256:b9d49ec4a75cf7457e39f49d974e93a9aef6c6cec6c2bfede92c3b7c8223f221
```

On each worker node, we'll run the `kubeadm join` command.

```
ubuntu@imaster00:~$ kubectl get nodes -o wide
NAME        STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION     CONTAINER-RUNTIME
imaster00   Ready    control-plane   68m   v1.30.5   10.0.0.12   <none>        Ubuntu 24.04 LTS   6.8.0-31-generic   containerd://1.7.22
inode00     Ready    <none>          98s   v1.30.5   10.0.0.13   <none>        Ubuntu 24.04 LTS   6.8.0-31-generic   containerd://1.7.22
inode01     Ready    <none>          88s   v1.30.5   10.0.0.14   <none>        Ubuntu 24.04 LTS   6.8.0-31-generic   containerd://1.7.22
```

And, after watching the pods intialize and come online, we can check our nodes again and see everything ready.

