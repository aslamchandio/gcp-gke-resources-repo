# GKE Standard Semi Private Zonal Clusters


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

## What this does?

This document shows how to create a Standard regional cluster to increase availability of the cluster's control plane and workloads during cluster upgrades, automated maintenance, or a zonal disruption.


## Single-zone versus multi-zonal
A single-zone cluster has a single control plane running in one zone. This control plane manages workloads on nodes running in the same zone. If you run a workload in a single zone, this workload is unavailable in the event of a zonal outage.

A multi-zonal cluster's nodes run in multiple zones, but it has only a single replica of the control plane. If you run a workload in multiple zones and there is a zonal outage, the workload is disrupted in that zone but remains available in other zones.

If you need higher availability for the control plane, consider creating a regional cluster instead. In a regional cluster, the control plane is replicated across multiple zones in a region.

After you create a cluster, you cannot change it from zonal to regional, or regional to zonal.

## Create a zonal cluster

The minimum information that you need to specify when creating a new zonal cluster is a name, project (usually the current project), and zone (usually the default location for command line tools), using the default settings for all other values. However, there are more possible configuration settings, only some of which are described in this section and some of which can't be changed after cluster creation. Ensure that you understand which settings can't be changed after cluster creation, and that you choose the right setting when creating a cluster if you don't want to have to create it again.

You can see an overview of cluster configuration options in About cluster configuration choices, and a complete list of possible options in the gcloud container clusters create and Terraform google_container_cluster reference guides.

### Semi Private Cluster 

```
gcloud config set accessibility/screen_reader false
gcloud config set compute/region us-central
gcloud config set compute/zone us-central1-c

```

```

gcloud container clusters create private-cluster1 \
    --region us-central1-c \
    --node-locations us-central1-b,us-central1-c \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.51.100.186/32 \
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
    --machine-type e2-medium \
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
gcloud container clusters describe public-cluster1 --zone us-central1-c

```

#### Deescribe Cluster Config File

```

gcloud container clusters describe private-cluster1 \
   --location=us-central1-c \
   --format="yaml(network, privateClusterConfig)"

```   

#### Check master-authorized-networks 

```

gcloud container clusters describe private-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --zone us-central1-c  

```

#### Add master-authorized-networks  IPS

```
gcloud container clusters update private-cluster1 \
    --zone us-central1-c \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.45.110.82/32,172.21.1.0/24

gcloud container clusters describe private-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --zone us-central1-c      

```

#### Connect GKE Cluster

```
gcloud container clusters list

gcloud container clusters get-credentials private-cluster1 --zone us-central1-c --project dev-project-786111 --internal-ip  

```

### Semi Private Cluster in other region

```

gcloud container clusters create private-cluster2 \
    --zone us-west1-a \
    --node-locations us-west1-a,us-west1-b \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.51.100.186/32 \
    --private-endpoint-subnetwork k8s-vpc-master-sub2-us-west1 \
    --network k8s-vpc \
    --subnetwork  k8s-vpc-sub2-us-west1  \
    --cluster-secondary-range-name pod-cidr1 \
    --services-secondary-range-name service-cidr \
    --enable-private-nodes \
    --enable-ip-alias \
    --enable-master-global-access \
    --gateway-api standard \
    --enable-dns-access  \
    --enable-ip-access \
    --machine-type e2-medium \
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

### Semi Private Cluster with custom master Cidr

```

gcloud container clusters create private-cluster3 \
    --region us-central1-f \
    --node-locations us-central1-b,us-central1-f \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.51.100.186/32 \
    --master-ipv4-cidr 172.16.2.0/28 \
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
    --machine-type e2-medium \
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

Note :  --master-ipv4-cidr 172.16.2.0/28

### Delete Clusters

```
gcloud container clusters list

gcloud container clusters delete private-cluster1 --zone us-central1-c
gcloud container clusters delete private-cluster2 --zone us-west1-a
gcloud container clusters delete private-cluster3 --zone us-central1-f

```



