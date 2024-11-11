# GKE Standard Public Zonal Clusters


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

You can use the gcloud CLI

### Public Cluster 

```
gcloud config set accessibility/screen_reader false
gcloud config set compute/region us-central
gcloud config set compute/zone us-central1-c

```

```
gcloud container clusters create public-cluster1 \
    --zone us-central1-f \
    --node-locations us-central1-a,us-central1-f \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.45.110.82/32 \
    --private-endpoint-subnetwork k8s-vpc-master-sub1-us-central1 \
    --network k8s-vpc \
    --subnetwork k8s-vpc-sub1-us-central1  \
    --cluster-secondary-range-name pod-cidr1 \
    --services-secondary-range-name service-cidr \
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

3- COMPUTE_ZONE: the zone for your node pool, such as us-central1-a.
The zone must be in the same region as the cluster control plane.

4- CHANNEL: the type of release channel, which can be one of rapid, regular, stable, or None. By default, the cluster is enrolled in the regular release channel unless at least one of the following flags is specified: --cluster-version, --release-channel, --no-enable-autoupgrade, and --no-enable-autorepair.
VERSION: the version you want to specify for your cluster.

#### Get Cluster Detail

```

gcloud container clusters list
gcloud container clusters describe public-cluster1 --zone us-central1-f

```

#### Deescribe Cluster Config File

```

gcloud container clusters describe public-cluster1 \
   --location=us-central1-f \
   --format="yaml(network, privateClusterConfig)"

```   

#### Check master-authorized-networks 

```
gcloud container clusters describe public-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --zone us-central1-f  

```

#### Add master-authorized-networks  IPS

```
gcloud container clusters update public-cluster1 \
    --zone us-central1-f \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.45.110.82/32,172.21.1.0/24

gcloud container clusters describe public-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --zone us-central1-f 

```

#### Check Pod Cidr on Nodes

```
kubectl get nodes gke-public-cluster1-default-pool-6ce20222-dxs3 -o yaml | grep CIDR

kubectl get nodes gke-public-cluster1-default-pool-c88fa138-3437 -o yaml | grep CIDR
      
```

### Public Cluster in other region

```
gcloud container clusters create public-cluster2 \
    --zone us-west1-a \
    --node-locations us-west1-a,us-west1-b \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.45.110.82/32 \
    --private-endpoint-subnetwork k8s-vpc-master-sub2-us-west1 \
    --network k8s-vpc \
    --subnetwork k8s-vpc-sub2-us-west1  \
    --cluster-secondary-range-name pod-cidr1 \
    --services-secondary-range-name service-cidr \
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

### Delete Clusters

```
gcloud container clusters list

gcloud container clusters delete public-cluster1 --zone us-central1-f
gcloud container clusters delete public-cluster2 --region us-west1-a

```



