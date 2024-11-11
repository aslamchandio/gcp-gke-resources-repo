# GKE Standard Public Regional Clusters


## What this does?

This document shows how to create a Standard regional cluster to increase availability of the cluster's control plane and workloads during cluster upgrades, automated maintenance, or a zonal disruption.

## Overview
When you create a regional cluster instead of a zonal cluster, the cluster's control plane is replicated across multiple zones in a given region. For node pools in a regional cluster, you can manually specify the zone(s) in which to run the node pools or you can use the default configuration, which replicates each node pool across three zones of the control plane's region. All zones must be within the same region as the cluster's control plane.

Regional clusters replicate resources across multiple zones and consume additional quotas.
After you create a regional cluster, you cannot convert it to a zonal cluster.

## Create a regional cluster with a single-zone node pool
The following instructions show you how to create a regional cluster with a node pool operating in a single zone within the region. The cluster's control plane is replicated across multiple zones in the specified region, but the nodes are located in the single zone, and are not replicated to other zones.

You can use the gcloud CLI,

### Public Cluster 

```
gcloud config set accessibility/screen_reader false
gcloud config set compute/region us-central
gcloud config set compute/zone us-central1-c

```

```
gcloud container clusters create public-cluster1 \
    --region us-central1 \
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
3- COMPUTE_ZONE: the zone for your node pool, such as us-central1-a. The zone must be in the same region as the cluster control plane.
4- CHANNEL: the type of release channel, which can be one of rapid, regular, stable, or None. By default, the cluster is enrolled in the regular release channel unless at least one of the following flags is specified: --cluster-version, --release-channel, --no-enable-autoupgrade, and --no-enable-autorepair.
VERSION: the version you want to specify for your cluster.

#### Get Cluster Detail

```

gcloud container clusters list
gcloud container clusters describe public-cluster1 --region us-central1

```

#### Deescribe Cluster Config File

```

gcloud container clusters describe public-cluster1 \
   --location=us-central1 \
   --format="yaml(network, privateClusterConfig)"

```   

#### Check master-authorized-networks 

```

gcloud container clusters describe public-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --region us-central1  

```

#### Add master-authorized-networks  IPS

```
gcloud container clusters update public-cluster1 \
    --region us-central1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.45.110.82/32,172.21.1.0/24

gcloud container clusters describe public-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --region us-central1      

```

#### Check Pod Cidr on Nodes

```
kubectl get nodes gke-public-cluster1-default-pool-6ce20222-dxs3 -o yaml | grep CIDR

kubectl get nodes gke-public-cluster1-default-pool-c88fa138-3437 -o yaml | grep CIDR
      
```

### Public Cluster in other region

```
gcloud container clusters create public-cluster2 \
    --region us-west1 \
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

gcloud container clusters delete public-cluster1 --region us-central1
gcloud container clusters delete public-cluster2 --region us-west1

```



