# GKE Standard Semi Private Regional Clusters


## How cluster network isolation works

In a GKE cluster, network isolation depends on who can access the cluster components and how. You can control:

Control plane access: You can customize external access, limited access, or unrestricted access to the control plane.
Cluster networking: You can choose who can access the nodes in Standard clusters, or the workloads in Autopilot clusters.
Before you create your cluster, consider the following:

Who can access the control plane and how is the control plane exposed?
How are your nodes or workloads exposed?
To answer these questions, follow the plan and design guidelines in About network isolation.

This document shows how to create a Standard regional cluster to increase availability of the cluster's control plane and workloads during cluster upgrades, automated maintenance, or a zonal disruption.


## Control plane

You can add up to 50 authorized networks (allowed CIDR blocks) in a project . For more information, refer to Define the IP addresses that can access the control plane.

While GKE can detect overlap with the control plane address block, it cannot detect overlap within a Shared VPC network.

## Cluster networking

Internal IP addresses for nodes come from the primary IP address range of the subnet you choose for the cluster. Pod IP addresses and Service IP addresses come from two subnet secondary IP address ranges of that same subnet. For more information, see IP ranges for VPC-native clusters.

GKE supports any internal IP address ranges, including private ranges (RFC 1918 and other private ranges) and privately used external IP address ranges. See the VPC documentation for a list of valid internal IP address ranges.

If you expand the primary IP range of a subnet to accommodate additional nodes, then you must add the expanded subnet's primary IP address range to the list of authorized networks for your cluster. If you don't, ingress-allow firewall rules relevant to the control plane aren't updated, and new nodes created in the expanded IP address space won't be able to register with the control plane. This can lead to an outage where new nodes are continuously deleted and replaced. Such an outage can happen when performing node pool upgrades or when nodes are automatically replaced due to liveness probe failures.

All nodes in a cluster with only private nodes are created without an external IP; they have limited access to Google Cloud APIs and services. To provide outbound internet access for your private nodes, you can use Cloud NAT.

Private Google Access is enabled automatically when you create a cluster unless you are using Shared VPC. You must not disable Private Google Access unless you are using NAT to access the internet.

### Semi Private Cluster 

```
gcloud config set accessibility/screen_reader false
gcloud config set compute/region us-central
gcloud config set compute/zone us-central1-c

```

```

gcloud container clusters create private-cluster1 \
    --region us-central1 \
    --node-locations us-central1-f \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.51.99.11/32,172.22.4.10/32,172.22.2.20/32 \
    --private-endpoint-subnetwork k8s-vpc-master-sub1-us-central1 \
    --network k8s-vpc \
    --subnetwork k8s-vpc-sub1-us-central1  \
    --cluster-secondary-range-name pod-cidr1 \
    --services-secondary-range-name service-cidr \
    --enable-private-nodes \
    --enable-ip-alias \
    --enable-master-global-access \
    --gateway-api standard \
    --enable-dns-access  \
    --enable-ip-access \
    --machine-type e2-small \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,GcpFilestoreCsiDriver \
    --disk-type pd-balanced  \
    --disk-size 50 \
    --default-max-pods-per-node 110 \
    --enable-dataplane-v2 \
    --enable-autorepair \
    --max-surge-upgrade 1 \
    --max-unavailable-upgrade 0 \
    --no-enable-basic-auth \
    --workload-pool dev-project-786111.svc.id.goog \
    --no-issue-client-certificate


```

1- CLUSTER_NAME: the name of your new regional cluster.
2- COMPUTE_REGION: the region for your cluster, such as us-central1.
3- COMPUTE_ZONE: the zone for your node pool, such as us-central1-a. The zone must be in the same region as the cluster control plane.
4- CHANNEL: the type of release channel, which can be one of rapid, regular, stable, or None. By default, the cluster is enrolled in the regular release channel unless at least one of the following flags is specified: --cluster-version, --release-channel, --no-enable-autoupgrade, and --no-enable-autorepair.
VERSION: the version you want to specify for your cluster.

You can configure the IP addresses that can access the control plane external and internal endpoints by using the following flags:

enable-master-authorized-networks: Specifies that access to the external endpoint is restricted to IP address ranges that you authorize.

master-authorized-networks: Lists the CIDR values for the authorized networks. This list is comma-delimited list. For example, 8.8.8.8/32,8.8.8.0/24.

enable-authorized-networks-on-private-endpoint: Specifies that access to the internal endpoint is restricted to IP address ranges that you authorize with the enable-master-authorized-networks flag.

no-enable-google-cloud-access: Denies access to the control plane from Google Cloud external IP addresses.

enable-master-global-access: Allows access from IP addresses in other Google Cloud regions.

#### Get Cluster Detail

```

gcloud container clusters list
gcloud container clusters describe private-cluster1 --region us-central1

```

#### Deescribe Cluster Config File

```

gcloud container clusters describe private-cluster1 \
   --location=us-central1 \
   --format="yaml(network, privateClusterConfig)"

```   

#### Check master-authorized-networks 

```

gcloud container clusters describe private-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --region us-central1  

```

#### Add master-authorized-networks  IPS

```
gcloud container clusters update private-cluster1 \
    --region us-central1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.45.110.82/32,172.21.1.0/24

gcloud container clusters describe private-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --region us-central1      

```

#### Connect GKE Cluster

```

gcloud container clusters list

gcloud container clusters get-credentials private-cluster1 --region us-central1 --project dev-project-786111 --internal-ip 

gcloud container node-pools list --cluster private-cluster1  --region us-central1

```

#### Workload on GKE

```

kubectl create deployment first-deployment --image aslam24/nginx-web-farmfresh:v1 --replicas 2

kubectl expose deployment first-deployment --type LoadBalancer --port=80 --target-port=80 --name first-deployment-lb-service    

```

#### Create Custom NodePool in GKE Private Cluster

```
gcloud container node-pools list --cluster private-cluster1  --region us-central1

gcloud container node-pools create nodepool-01 \
  --cluster private-cluster1 \
  --region us-central1 \
  --node-locations us-central1-f,us-central1-c \
  --machine-type e2-medium \
  --disk-type pd-balanced  \
  --disk-size 40 \
  --num-nodes 1

  gcloud container node-pools list --cluster private-cluster1  --region us-central1
  gcloud container node-pools describe nodepool-01 --cluster private-cluster1  --region us-central1

```

#### Drain and Cordon Default Node

```

for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
   kubectl cordon "$node";
done
   
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
   kubectl drain --force --ignore-daemonsets --delete-local-data --grace-period=10 "$node";
done

kubectl get nodes
kubectl get nodes -o wide

```

#### Delete Default Node

```
gcloud container node-pools list --cluster private-cluster1  --region us-central1

kubectl get nodes
kubectl get nodes -o wide

gcloud container node-pools delete default-pool \
  --cluster private-cluster1 \
  --region us-central1

gcloud container node-pools list --cluster private-cluster1  --region us-central1

kubectl get nodes
kubectl get nodes -o wide


```

#### Get Pods CIDRs Ranges in Nodes

```
kubectl get nodes gke-private-cluster1-nodepool-01-4284cc98-zr67 -o yaml | grep CIDR

 podCIDR: 10.244.2.0/24
  podCIDRs:


kubectl get nodes gke-private-cluster1-nodepool-01-81cfea3d-24nw -o yaml | grep CIDR

podCIDR: 10.244.1.0/24
  podCIDRs:
```

### Scale a NodePool nodepool-01

#### Horizontally scale by changing the node count

```

gcloud container clusters resize private-cluster1 \
    --node-pool nodepool-01 \
    --region us-central1  \
    --num-nodes 2

gcloud container node-pools list --cluster private-cluster1  --region us-central1
gcloud container node-pools describe nodepool-01 --cluster private-cluster1  --region us-central1

kubectl get nodes
kubectl get nodes -o wide

```

#### Vertically scale by changing the node machine attributes

```

  gcloud container node-pools update nodepool-01 \
    --cluster private-cluster1 \
    --region us-central1  \
    --machine-type e2-small \
    --disk-type pd-balanced \
    --disk-size 50

kubectl get nodes  -w (on another terminal)
kubectl get pods   -w (on another terminal)

gcloud container node-pools list --cluster private-cluster1  --region us-central1
gcloud container node-pools describe nodepool-01 --cluster private-cluster1  --region us-central1

kubectl get nodes
kubectl get nodes -o wide

```

### Autoscaling a Cluster in GKE Private Cluster

```
gcloud container clusters update private-cluster1 \
    --enable-autoscaling \
    --node-pool  nodepool-01 \
    --min-nodes 1  \
    --max-nodes 2 \
    --region us-central1

gcloud container node-pools list --cluster private-cluster1  --region us-central1
gcloud container node-pools describe nodepool-01 --cluster private-cluster1  --region us-central1

kubectl get nodes
kubectl get nodes -o wide

```

#### Create Custom NodePool in GKE Private Cluster

```
gcloud container node-pools create nodepool-02 \
  --cluster private-cluster1 \
  --region us-central1 \
  --node-locations us-central1-a \
  --machine-type e2-small \
  --disk-type pd-balanced  \
  --disk-size 30 \
  --num-nodes 1

gcloud container node-pools list --cluster private-cluster1  --region us-central1
gcloud container node-pools describe nodepool-02 --cluster private-cluster1  --region us-central1

```

# Autoscaling a Cluster in GKE Private Cluster

```
gcloud container clusters update private-cluster1 \
    --enable-autoscaling \
    --node-pool  nodepool-02 \
    --min-nodes 1  \
    --max-nodes 2 \
    --region us-central1

gcloud container node-pools list --cluster private-cluster1  --region us-central1
gcloud container node-pools describe nodepool-02 --cluster private-cluster1  --region us-central1

kubectl get nodes
kubectl get nodes -o wide

```

#### Drain and Cordon nodepool-01

```

for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=nodepool-01 -o=name); do
   kubectl cordon "$node";
done
   
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=nodepool-01 -o=name); do
   kubectl drain --force --ignore-daemonsets --delete-local-data --grace-period=10 "$node";
done

kubectl get nodes
kubectl get nodes -o wide

```

#### Delete nodepool-01

```
gcloud container node-pools list --cluster private-cluster1  --region us-central1

kubectl get nodes
kubectl get nodes -o wide

gcloud container node-pools delete nodepool-01 \
  --cluster private-cluster1 \
  --region us-central1

gcloud container node-pools list --cluster private-cluster1  --region us-central1

kubectl get nodes
kubectl get nodes -o wide


```

 #### Connect GKE Cluster using GCP VM from Others Regions

```

gcloud container clusters list

gcloud container clusters get-credentials private-cluster1 --region us-central1 --project dev-project-786111 --internal-ip    

```

#### Connect GKE Private Node using Putty

```

gcloud compute instances list

gcloud compute ssh gke-private-cluster1-nodepool-02-9682b92e-64n9 --zone us-central1-a  --tunnel-through-iap --project dev-project-786111

gcloud compute start-iap-tunnel gke-private-cluster1-nodepool-02-9682b92e-64n9 22 --zone us-central1-a --local-host-port localhost:2244  --project dev-project-786111

sudo crictl ps

sudo crictl ps | grep first-deployment

sudo crictl logs -f 297974ae5e513

```

### Delete Clusters

```
gcloud container clusters list

gcloud container clusters delete private-cluster1 --region us-central1

```



