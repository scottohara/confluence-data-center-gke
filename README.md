### Stand up a 4-node Confluence Data Center cluster using Google Kubernetes Engine

Note
====
In it's current form, with this set of k8s configs it is currently not possible to run more than 2 nodes.
The limitation is due to the way Data Center nodes discover each other at startup.

Confluence Data Center supports the following options for node discovery:
1. TCP/IP - assumes that you know the IP addresses of all nodes in advance (which defeats the point of ephemeral pods in Kubernetes
2. Multicast - no public Kubernetes-as-a-service platform supports multicast networking by default
3. AWS - only useful in AWS

# Please do not use this for a production Confluence Data Center cluster. This repo is intended for dev/test use cases only.

Alternatively, for a simpler setup, you may want to consider running [Confluence Data Centre in Docker](https://github.com/scottohara/confluence-data-center-docker).

Overview
=======
1. Create a new Kubernetes cluster in GKE
2. Deploy a single-node file server (to use as the shared home file system)
3. Deploy k8s resources necessary to run a postgres database
4. Deploy a load balancer service
5. Deploy the first Confluence Data Center node
6. Complete the first run setup (time bomb license, db setup etc.)
7. Scaling up to a second node
8. Scaling up to a third node (NOTE: currently doesn't work)
9. Scaling up to a fourth node (NOTE: currently doesn't work)
10. Cleaning up

Step-by-step
===========

### Create a new Kubernetes cluster in GKE

(Assumes you already have a Google Cloud Platform account & project created, and you have the `gcloud` CLI installed)

```bash
gcloud container clusters create confluence-data-center
```

The above command will create a new standard Kubernetes cluster called `confluence-data-center` using all defaults.
At the time of writing, this means a 3-node, 1 x vCPU, 3.75Gb RAM, Kubernetes v1.10 cluster in the `us-central1-a` zone.

The command also automatically reconfigures your `kubectl` to point to the newly created cluster.

### Deploy a single-node file server

Most public Kubernetes-as-a-service platforms don't support shared, writable filesystems (i.e. a volume that can be mounted by multiple running pods, and allow all pods to write to it); or if they do, they're very expensive.

A cheaper option is to run a [single-node NFS server](https://console.cloud.google.com/marketplace/details/click-to-deploy-images/singlefs).

To save costs, reduce the CPU to 1 x vCPU; and make sure it is created in the same zone as your Kubernetes cluster (`us-central1-a`)

The deployment will take a few minutes to finish. Once it's deployed, you'll need to get the IP address of the NFS server (so that you can point your Confluence pods at it), so run the following command:

```bash
gcloud compute instances describe singlefs-1-vm --format='value(networkInterfaces[0].networkIP)'
```

1. Copy the IP address returned by the above command
2. Open the file `confluence/statefulset.yaml`
3. Near the bottom, find the `shared-home` NFS volume
4. Paste the IP address into the `server` value

```yaml
volumes:
 - name: shared-home
   nfs:
    path: "/data"
    server: PASTE IP ADDRESS HERE
```

### Deploy k8s resources necessary to run a postgres database

The YAML files in the `database` folder can be applied with a single command:

```bash
kubectl apply -f database
```

* `config.yaml` is used to configure the name of the database, the admin password, and the location of the data files.
* `storage.yaml` defines a writable persistent volume for storing data.
* `service.yaml` creates a `ClusterIP` service, allowing each Confluence node to access the database on port 5432.
* `deployment.yaml` creates a single instance of postgres 11, using the persistent volume above. 

### Deploy a load balancer service

A service of type `LoadBalancer` sits in front of the Confluence nodes, and allows access to the cluster on `http://<external IP address>`

For testing purposes, it can be useful to ensure that your Confluence session remains on the same Confluence node, so we use `sessionAffinity: ClientIP` to stick sessions to the node that they started on.

To create the load balancer, run the following command:

```bash
kubectl apply -f confluence/service.yaml
```

It will take a few minutes for GKE to allocate an external IP address. Once the external IP address is available, you'll need to update the `CATALINA_CONNECTOR_PROXYNAME` environment variable with it.

Periodically running the following command until an external IP address is displayed:

```bash
kubectl get service/confluence
```

1. Copy the external IP address shown
2. Open the file `confluence/config.yaml`
3. Paste the IP address into the `CATALINA_CONNECTOR_PROXYNAME` value

```yaml
data:
  CATALINA_CONNECTOR_PROXYNAME: PASTE IP ADDRESS HERE
  CATALINA_CONNECTOR_PROXYPORT: "80"
```

### Deploy the first Confluence Data Center node

The first Confluence node can be created with a single command:

```bash
kubectl apply -f confluence/config.yaml -f confluence/statefulset.yaml
```

It will take a few minutes for GKE to initialise and run the pod, so keep checking periodically with the following command until the pod is ready:

```bash
kubectl get pods -o wide
```

### Complete first run setup (time bomb license, db setup etc.)

Point your web browser at http://<external IP address>, and you should (hopefully) see the Confluence setup page.

When prompted for a license, if you don't have a valid Confluence Data Center license (or an active trial license); you can use a [timebomb license](https://developer.atlassian.com/platform/marketplace/timebomb-licenses-for-testing-server-apps/). Find "10 user Confluence Data Center license, expires in 3 hours" and copy/paste the license key.

Give your cluster an appropriate name.

When prompted for how nodes will discover each other, choose the TCP option and enter the IP address of your first Confluence pod (which you can find in the output of the `kubectl get pods -o wide` command from earlier).

For the database setup, the server/address is `postgres`, the port is 5432, the database/user name is `confluence` and the password is `admin123` (unless you changed these values in `database/config.yaml`).

Continue through the setup process until the end. If all went well, you should have a fully functioning (albeit single node) Confluence Data Center instance running. You can check that it is a Data Center instance by going to the Admin section and scrolling to down to the Cluster details.

### Scaling up to a second node

The second (and subsequent) Confluence nodes need a copy of the configuration from the initial node.

The pod configuration is setup to look for a `confluence.cfg.xml.template` file in the shared home, so the first step is to copy the config from the first node, using the following commands:

```bash
kubectl exec confluence-0 -- cp /var/atlassian/application-data/confluence/confluence.cfg.xml /var/atlassian/application-data/shared/confluence.cfg.xml.template
kubectl exec confluence-0 -- chmod 777 /var/atlassian/application-data/shared/confluence.cfg.xml.template
```

Finally, to scale to a second node; run the following command:

```bash
kubectl scale --replicas=2 statefulset/confluence
```

Once again, keep an eye on the progress of the new pod using `kubectl get pods -o wide` until it is ready.

Then, back in the Confluence Admin page, check the Clusters page again and you should (hopefully) see the second node appear.

# From this point on, the instructions for scaling to a 3rd or 4th node do not currently work.

You will get a 3rd/4th node, but one of the initial nodes will instantly die; and an endless loop of pods starting and other pods terminating will occur.

### Scaling up to a third node (NOTE: currently doesn't work)

Open an SSH session to your single-node NFS server (via Cloud Console), and update the `confluence.cluster.peers` property in `confluence.cfg.xml.template` with the IP address of the `confluence-1` pod.

Then scaling the cluster to a third node is the same as the second node, e.g.

```bash
kubectl scale --replicas=3 statefulset/confluence
```

### Scaling up to a fourth node (NOTE: currently doesn't work)

Open an SSH session to your single-node NFS server (via Cloud Console), and update the `confluence.cluster.peers` property in `confluence.cfg.xml.template` with the IP address of the `confluence-2` pod.

Then scaling the cluster to a fourth node is the same as the second/third nodes, e.g.

```bash
kubectl scale --replicas=4 statefulset/confluence
```

### Cleaning up

After you have finished, you can completely tear down your cluster as follows:

```bash
gcloud container clusters delete confluence-data-center
gcloud deployment-manager deployments delete singlefs-1
```

This completely removes the Kubernetes cluster (including the Confluence nodes, the Postgres database, the load balancer, the persistent volumes etc.), and the NFS server.