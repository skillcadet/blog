---
layout: post
title: Setup a Edge Loadbalancer using NGINX and NodePort
---

We'll begin the process of bringing up a private cluster by creating a host that exists on both the public network and an internal private network.

Create a new instance with an external ip address or that has external ports mapped for 22, 80 and 443.  Create an internal ip address on the same or a second interface.
Initialize the instance to include nginx and certbot using [cloud-init](https://github.com/iweave/xd7k8s/blob/main/cloud-init/cloud-init-ubuntu2404-proxy-xd7tower.yaml).
Now we provision the host in the cluster so we can get our initial services online. (This also establishes the internal ip address if you don't choose it in advance)


### SSH config

We're going to be making changes to the local ~/.ssh/config file to facilite communicating to our cluster.

```
Host xd7tower
    HostName 161.200.20.100
    User ubuntu
    IdentityFile ~/.ssh/id_rsa_blog_skillcadet
```

Now we ssh to the tower machine for the first time:

```
ssh xd7tower
The authenticity of host '161.200.20.100 (161.200.20.100)' can't be established.
ECDSA key fingerprint is SHA256:k6AjX1v7wDxj/QYffwiHdAKS8rR2cGIMTIQEben5UH8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '161.200.20.100' (ECDSA) to the list of known hosts.
```

We'll need to approve the host fingerprint, which we often need to do blindly unless we have a [secure console](https://blog.skillcadet.com/extras/verify-ssh-fingerprint.html) to the node/instance.

### DNS

Having established that our edge host is reachable, add the IP address to a registered domain name so that LetsEncyrpt can find it further on.

This entails adding the IPv4 A (and optionally IPv6 AAAA ) addresses to your DNS configuration.

```
161.200.20.100          IN A    blog.xd7.org.
2607:b500:200:2000:1::1  IN AAAA blog.xd7.org.
```

And then you'll need to wait a bit for DNS updates to propagate. This could take as long as 24 hours, but in practice is usually under an hour and can be faster. Test by using `nslookup blog.xd7.org` or `dig blog.xd7.org` to watch for your changes to appear.  Additions to the domain name may propagate faster than changes as some internet providers cache DNS results to improve performance.

Some DNS servers for large broadband carriers aggressively cache DNS information, so it may be beneficial to search for the official Nameserver to watch for updates to be published.

```
$ dig -t NS blog.xd7.org

; <<>> DiG 9.10.6 <<>> -t NS blog.xd7.org

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;blog.xd7.org.			IN	NS

;; AUTHORITY SECTION:
xd7.org.		300	IN	SOA	dns1.name-services.com. info.name-services.com. 1731517309 172800 900 1814400 300

```

Here we see a request for the Nameserver records. In the Autority Section we see two important pieces of information in the SOA record. The host domain `xd7.org` and the serial number of the DNS update `1731517309`.  One thing to note about the serial number, if it looks substantially like a unix Epoch time (common), you can convert that time to find when the records were last updated. Another common format is `YYYYMMDDXX`. All that's officially required is that the serial number has to always increment (roll overs occur on a 32 bit modulus).

```
$ dig -t NS xd7.org

; <<>> DiG 9.10.6 <<>> -t NS xd7.org

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;xd7.org.			IN	NS

;; ANSWER SECTION:
xd7.org.		0	IN	NS	dns3.name-services.com.
xd7.org.		0	IN	NS	dns5.name-services.com.
xd7.org.		0	IN	NS	dns1.name-services.com.
xd7.org.		0	IN	NS	dns2.name-services.com.
xd7.org.		0	IN	NS	dns4.name-services.com.

;; ADDITIONAL SECTION:
dns5.name-services.com.	1240	IN	A	64.98.148.139
dns5.name-services.com.	1502	IN	AAAA	2604:4000:2800:2000:64:98:148:139
dns1.name-services.com.	293	IN	A	64.98.148.137
dns1.name-services.com.	317	IN	AAAA	2604:4000:2800:2000:64:98:148:137
dns2.name-services.com.	898	IN	A	216.40.47.201
dns2.name-services.com.	676	IN	AAAA	2604:4000:0:d:216:40:47:201
dns4.name-services.com.	3140	IN	A	216.40.47.202
dns4.name-services.com.	710	IN	AAAA	2604:4000:0:d:216:40:47:202
dns3.name-services.com.	1209	IN	A	64.98.148.138
dns3.name-services.com.	1502	IN	AAAA	2604:4000:2800:2000:64:98:148:138
```

Now we can request the NS record of the root domain and get the official nameservers, which we can add to our DNS probes.

We can either watch for the serial number to update or keep querying the target domain name, while specifying to use one of the official nameservers for the lookup.

 `nslookup blog.xd7.org dns1.name-services.com.` or `dig blog.xd7.org @dns1.name-services.com` 

### Nginx 

Now we need to build an Nginx seed configuration file for LetsEncrypt to manage.

/etc/nginx/sites-available/blog.xd7.org.conf
```
upstream blog_xd7_backends {
    server inode00:30081;
    server inode01:30081;
}

server {
        listen 80;
        listen [::]:80;
        server_name blog.xd7.org;

        location / {
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_pass http://blog_xd7_backends;
        }
}
```

Here, we add an `upstream` stanza, pointing to each of our available nodes and the NodePort for our deployment. You could create this file in your cloud-init using `write-files:` stanza, customized to your site.

```
sudo ln -s /etc/nginx/sites-available/blog.xd7.org.conf /etc/nginx/sites-enabled
sudo nginx -t
  nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
  nginx: configuration file /etc/nginx/nginx.conf test is successful
sudo systemctl reload nginx
```

And, now, to activate the site, we need to link the configuration file to the active directory, test that our configuration is good and reload the nginx configuration.

### Certbot - Let's Encrypt

Our example application, [Ghost](https://ghost.org), requires HTTPS, which is a recommended for most sites.

We will accomplish this by using the Free services from [Let's Encrypt](https://letsencrypt.org) using the ACME certbot which we preinstalled with cloud-init.

```
sudo certbot --nginx -d blog.xd7.org
```
Still on the tower system, request an SSL certificate for our site.  You'll be asked to provide your email address for administrative purposes, and if you want to have your email address subscribed to the [EFF](https://www.eff.org) announcements list.

### Profit

Now, we can finally browse to our exposed service, using secure HTTPS, through the Nginx Reverse Proxy.

[https://blog.xd7.org](https://blog.xd7.org)

