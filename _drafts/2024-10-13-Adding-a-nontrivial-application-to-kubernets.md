---
layout: post
title: Adding a nontrivial application to Kubernets
---

In the preceeding posts in this series, we've [bootstrapped hosts with cloud-config](one) and [initiallized a multi-node, multi-network kubernetes cluster](two).

Continuing on, our next step is to launch an application into kubernetes.  There are lots of tutorials about launching simple stateless containers, I wanted to tackle something a bit meatier to show how you could launch more complicated environments.

## Ghost on Kubernetes.

The application targetted for this tutorial is the blog software [Ghost](https://www.ghost.org).  Ghost is a feature rich blog application that I find easier to use and maintain than wordpress. Ghost uses MySQL 8 as it backend. This gives us practice on setting up persistent storage on the cluster and external load balancing to cluster services.

While investigating what currently exists for this solution, I came across [this blog](https://github.com/sredevopsorg/ghost-on-kubernetes), and I will initially be using some of the configuration for the ghost portion of this post. However, for the MySQL installation, much like we used the operator to deploy Calico in the last post, we'll be using the offical MySQL operator to setup or database.

### MySQL Operator
```
helm repo add mysql-operator https://mysql.github.io/mysql-operator/
helm repo update
helm install my-mysql-operator mysql-operator/mysql-operator \
   --namespace mysql-operator --create-namespace
```

We're going to be using [Helm](https://helm.sh/docs/intro/install/) to install our MySQL operator, Helm installation instructions can be found in the link.
First, we add the repository to Helm, update Helm to pull down the config and then install the operator.


## Code repository
Follow along by cloneing or forking [this repository](xd7org) and editing the following files.

00-namespace.yaml
```
apiVersion: v1
kind: Namespace
metadata:
  name: ghost-on-kubernetes
  labels:
    app: ghost-on-kubernetes
    app.kubernetes.io/name: ghost-config-prod
    app.kubernetes.io/instance: ghost-on-kubernetes
    app.kubernetes.io/version: '5.92'
    app.kubernetes.io/component: ghost-config
    app.kubernetes.io/part-of: ghost-on-kubernetes
```

Here we want to isolate our application by using a namespace. The database can be in the same namespace or different namespaces from the application. In this example, we'll deploy both into ghost-on-kubernetes namespace.

01-local-pv-mysql.yaml
```
kind: PersistentVolume
metadata:
  name: local-pv-mysql
spec:
  capacity:
    storage: 80Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/pv/disk1/mysql
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - inode00
```

01-local-pv-ghost.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-ghost
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/pv/disk1/ghost
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - inode00
                - inode01
```

For persistent storage on our cluster, we need to define a couple of PersistenVolumes. On node00, I added a 100GB disk mounted at /mnt/pv/disk1. Because the root of a mounted volume contains a lost+found directory, we need to create an empty folder or mysql will refuse to create a database. `sudo mkdir -p /mnt/pv/disk1/mysql`. We'll also create a pv folder for ghost on either node, even though we can only install ghost on one node at at time (unless we use a technology like [Rook.io](https://www.rook.io) that gives us ReadWriteMany volumes).

02-local-pvc-mysql.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: datadir-mycluster-0
  namespace: ghost-on-kubernetes
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 80Gi
  storageClassName: local-storage
```

02-local-pvc-ghost.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: k8s-ghost-content
  namespace: ghost-on-kubernetes
  labels:
    app: ghost-on-kubernetes
    app.kubernetes.io/name: k8s-ghost-content
    app.kubernetes.io/instance: ghost-on-kubernetes
    app.kubernetes.io/version: '5.92'
    app.kubernetes.io/component: storage
    app.kubernetes.io/part-of: ghost-on-kubernetes

spec:
  # Change this to your storageClassName, we suggest using a storageClassName that supports ReadWriteMany for production.
  storageClassName: "local-storage"
  volumeMode: Filesystem
  # Change this to your accessModes. We suggest ReadWriteMany for production, ReadWriteOnce for development.
  # With ReadWriteMany, you can have multiple replicas of Ghost, so you can achieve high availability.
  # Note that ReadWriteMany is not supported by all storage providers and may require additional configuration.
  # Ghost officialy doesn't support HA, they suggest using a CDN or caching. Info: https://ghost.org/docs/faq/clustering-sharding-multi-server/
  accessModes:
  - ReadWriteOnce # Change this to your accessModes if needed, we suggest ReadWriteMany so we can scale the deployment later.
  resources:
    requests:
      storage: 10Gi
```

Next we setup PersistentVolumeClaims that our containers will use. In this on-premise cluster, we don't support ReadWriteMany access mode, so we stick with the default of ReadWriteOnce.

03-mysql-secrets.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: mypwds
  namespace: ghost-on-kubernetes
stringData:
  rootUser: root
  rootHost: '%'
  rootPassword: Bl0gxd7@My5ql
```
Secrets are next, we begin with the mysql secrets describing the mysql Root user. We'll need this user to configure our database.

04-ghost-config.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: ghost-config-prod
  namespace: ghost-on-kubernetes
  labels:
    app: ghost-on-kubernetes
    app.kubernetes.io/name: ghost-config-prod
    app.kubernetes.io/instance: ghost-on-kubernetes
    app.kubernetes.io/version: '5.92'
    app.kubernetes.io/component: ghost-config
    app.kubernetes.io/part-of: ghost-on-kubernetes
type: Opaque
stringData:
  config.production.json: |-
    {
      "url": "https://blog.xd7.org",
      "admin": {
        "url": "https://blog.xd7.org"
      },
      "server": {
        "port": 2368,
        "host": "0.0.0.0"
      },
      "mail": {
        "transport": "SMTP",
        "from": "xd7org@gmail.com",
        "options": {
          "service": "Google",
          "host": "smtp.gmail.com",
          "port": 465,
          "secure": true,
          "auth": {
            "user": "xd7org@gmail.com",
            "pass": "onea twob thre four"
          }
        }
      },
      "logging": {
        "transports": [
          "stdout"
        ]
      },
      "database": {
        "client": "mysql",
        "connection":
        {
          "host": "mycluster",
          "user": "nonroot",
          "password": "Bl0gxd7@My5qlUser",
          "database": "blogxd7org",
          "port": "3306"
        }
      },
      "process": "local",
      "paths": {
        "contentPath": "/home/nonroot/app/ghost/content"
      }
    }
```

Modified as from the repo, an update to this blog will replace the configuration file with secrets and configmap mapped to environment variables. Here we need to specify our blogs coniguration settings. Here we specify the url for the blog, the database connection settings and the email settings (for simplicity, an application password is configured for a gmail user). Email is necessary for registration and newsletter features. This is simple, but requires two-factor authentication to enable. Since the configuration option is sometimes missing from the Google Security Page, after you have confirmed that 2FA is active, follow [this link](https://myaccount.google.com/apppasswords) to create an application password.

05-mysql-pod.yaml
```
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
  namespace: ghost-on-kubernetes
spec:
  secretName: mypwds
  tlsUseSelfSigned: true
  instances: 1
  version: 8.4.1
  router:
    instances: 1
    version: 8.4.1
  datadirVolumeClaimTemplate:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 80Gi
    storageClassName: local-storage
```

Now we actually will provision the mysql containers. Our secrets will be used to initialize the DB and we're requesting Mysql 8.4.1 (Recommended for Ghost), one node for the database, one node for the router and we've specified to find a 80GB `local-storage` PersistentVolumeClaim.


```
kubectl describe pods mycluster-0 -n ghost-on-kubernetes
....
  Normal  Logging    42s   kopf               Handler 'on_pod_create' succeeded.
  Normal  Logging    42s   kopf               Creation is processed: 1 succeeded; 0 failed.
  Normal  Logging    42s   kopf               create_cluster OK
```

This takes a few minutes to deploy, we can monitor the cluster coming up.

```
kubectl run -n ghost-on-kubernetes --rm -it myshell --image=container-registry.oracle.com/mysql/community-operator -- mysqlsh
If you don't see a command prompt, try pressing enter.

 MySQL  SQL > \connect root@mycluster
Creating a session to 'root@mycluster'
Please provide the password for 'root@mycluster': *************
Save password for 'root@mycluster'? [Y]es/[N]o/Ne[v]er (default No):
Fetching global names for auto-completion... Press ^C to stop.
Your MySQL connection id is 0
Server version: 8.4.1 MySQL Community Server - GPL
No default schema selected; type \use <schema> to set one.
```

Once the mysql cluster is started, we need to login to the database and create a regular user with full privileges on our intended database. We begin by launching a mysql container in the correct namespace, using `\connect root@mycluster` to connect to the cluster and then entering the root mysql password we configured in the mysql secrets `mypwds`.

```
CREATE USER 'nonroot'@'%' IDENTIFIED BY 'Bl0gxd7@My5qlUser';
GRANT ALL PRIVILEGES ON blogxd7org.* to 'nonroot'@'%';
```

Here we create the user and grant the privileges necessary to run the blog.  Press `Control+D` to end the mysql administrative session.

06-ghost-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost-on-kubernetes
  namespace: ghost-on-kubernetes
  labels:
    app: ghost-on-kubernetes
    app.kubernetes.io/name: ghost-on-kubernetes
    app.kubernetes.io/instance: ghost-on-kubernetes
    app.kubernetes.io/version: '5.92'
    app.kubernetes.io/component: ghost
    app.kubernetes.io/part-of: ghost-on-kubernetes


spec:
  # If you want HA for your Ghost instance, you can increase the number of replicas AFTER creation and you need to adjust the storage class. See 02-pvc.yaml for more information.
  replicas: 1
  selector:
    matchLabels:
      app: ghost-on-kubernetes
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 3
  revisionHistoryLimit: 4
  progressDeadlineSeconds: 600
  template:
    metadata:
      namespace: ghost-on-kubernetes
      labels:
        app: ghost-on-kubernetes
    spec:
      automountServiceAccountToken: false # Disable automounting of service account token
      volumes:
      - name: k8s-ghost-content
        persistentVolumeClaim:
          claimName: k8s-ghost-content

      - name: ghost-config-prod
        secret:
          secretName: ghost-config-prod
          defaultMode: 420

      - name: tmp
        emptyDir:
          sizeLimit: 64Mi

      initContainers:
      - name: permissions-fix
        imagePullPolicy: Always
        image: docker.io/busybox:stable-musl
        env:
        - name: GHOST_INSTALL
          value: /home/nonroot/app/ghost
        - name: GHOST_CONTENT
          value: /home/nonroot/app/ghost/content
        - name: NODE_ENV
          value: production
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
        resources:
          limits:
            cpu: 900m
            memory: 1000Mi
          requests:
            cpu: 100m
            memory: 128Mi
        command:
          - /bin/sh
          - '-c'
          - |
            set -e

            export DIRS='files logs apps themes data public settings images media'
            echo 'Check if base dirs exists, if not, create them'
            echo "Directories to check: $DIRS"
            for dir in $DIRS; do
              if [ ! -d $GHOST_CONTENT/$dir ]; then
                echo "Creating $GHOST_CONTENT/$dir directory"
                mkdir -pv $GHOST_CONTENT/$dir && chown -Rfv 65532:65532 $GHOST_CONTENT/$dir || echo "Error creating $GHOST_CONTENT/$dir directory"
              fi
            done

            echo "Changing ownership of $GHOST_CONTENT/themes directory"
            chown -Rf 65532:65532 $GHOST_CONTENT/themes && echo "chown ok on themes" || echo "Error changing ownership of $GHOST_CONTENT/themes directory"

            echo "Changing ownership of $GHOST_CONTENT/public directory"
            chown -Rf 65532:65532 $GHOST_CONTENT/public && echo "chown ok in public" || echo "Error changing ownership of $GHOST_CONTENT public directory"
            exit 0


        volumeMounts:
        - name: k8s-ghost-content
          mountPath: /home/nonroot/app/ghost/content
          readOnly: false

      containers:
      - name: ghost-on-kubernetes
        image: ghcr.io/sredevopsorg/ghost-on-kubernetes:main
        imagePullPolicy: Always
        ports:
        - name: ghk8s
          containerPort: 2368
          protocol: TCP

        # You should uncomment the following lines in production. Change the values according to your environment.
        readinessProbe:
          httpGet:
            path: /ghost/api/v4/admin/site/
            port: ghk8s
            httpHeaders:
            - name: X-Forwarded-Proto
              value: https
            - name: Host
              value: blog.xd7.org
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
          initialDelaySeconds: 10

        livenessProbe:
          httpGet:
            path: /ghost/api/v4/admin/site/
            port: ghk8s
            httpHeaders:
            - name: X-Forwarded-Proto
              value: https
            - name: Host
              value: blog.xd7.org
          periodSeconds: 300
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 1
          initialDelaySeconds: 30

        env:
        - name: NODE_ENV
          value: production
        resources:
          limits:
            cpu: 800m
            memory: 800Mi
          requests:
            cpu: 100m
            memory: 256Mi

        volumeMounts:
        - name: k8s-ghost-content
          mountPath: /home/nonroot/app/ghost/content
          readOnly: false
        - name: ghost-config-prod
          readOnly: true
          mountPath: /home/nonroot/app/ghost/config.production.json
          subPath: config.production.json
        - name: tmp # This is the temporary volume mount to allow loading themes
          mountPath: /tmp
          readOnly: false
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 65532


      restartPolicy: Always
      terminationGracePeriodSeconds: 15
      dnsPolicy: ClusterFirst
      # Optional: Uncomment the following to specify node selectors
      #affinity:
      #  nodeAffinity:
      #    requiredDuringSchedulingIgnoredDuringExecution:
      #      nodeSelectorTerms:
      #        - matchExpressions:
      #            - key: node-role.kubernetes.io/worker
      #              operator: In
      #              values:
      #                - 'true'
      securityContext: {}
```

Next, we deploy the ghost container. We need to modify the host name in the `readinessProbe` and `livenessProbe`.
This will take some time to deploy

07-ghost-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: ghost-on-kubernetes-service
  namespace: ghost-on-kubernetes
  labels:
    app: ghost-on-kubernetes
    app.kubernetes.io/name: ghost-on-kubernetes-service
    app.kubernetes.io/instance: ghost-on-kubernetes
    app.kubernetes.io/version: '5.92'
    app.kubernetes.io/component: service-frontend
    app.kubernetes.io/part-of: ghost-on-kubernetes


spec:
  ports:
  - port: 2368
    protocol: TCP
    targetPort: ghk8s
    name: ghk8s
    nodePort: 30081
  type: NodePort
  selector:
    app: ghost-on-kubernetes
```

Finally, we expose the the deployment using a nodePort, which we declare so that we can map to it with an external load balancer.

Connecting our nodePort exposed service using Nginx as a reverse proxy load balancer is covered in [this blog](edge).

Once we have deployed our load balancer and setup HTTPS, we can see our new blog at [https://blog.xd7.org](https://blog.xd7.org).  To gain administrative control on the blog, we begin by going to the admin URL [https://blog.xd7.org/ghost](https://blog.xd7.org/ghost). Here we fill in the basic information for our site and proceed into the administrative backend.
 
