# GKE Standard Private Zonal Clusters


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

### Private Cluster 

```
gcloud config set accessibility/screen_reader false
gcloud config set compute/region us-central
gcloud config set compute/zone us-central1-c

```
### Enable Private Endpoint

enable-private-endpoint: Specifies that access to the external endpoint is disabled. Omit this flag if you want to allow access to the control plane from external IP addresses. In this case, we strongly recommend that you control access to the external endpoint with the enable-master-authorized-networks flag.

```

gcloud container clusters create private-cluster1 \
    --zone us-central1-f \
    --node-locations us-central1-c,us-central1-f \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 172.22.4.10/32 \
    --private-endpoint-subnetwork k8s-vpc-master-sub1-us-central1 \
    --network k8s-vpc \
    --subnetwork k8s-vpc-sub1-us-central1  \
    --cluster-secondary-range-name pod-cidr1 \
    --services-secondary-range-name service-cidr \
    --enable-private-nodes \
    --enable-private-endpoint \
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

enable-private-endpoint: Specifies that access to the external endpoint is disabled. Omit this flag if you want to allow access to the control plane from external IP addresses. In this case, we strongly recommend that you control access to the external endpoint with the enable-master-authorized-networks flag.

#### Get Cluster Detail

```

gcloud container clusters list
gcloud container clusters describe private-cluster1 --zone us-central1-f

```

#### Deescribe Cluster Config File

```

gcloud container clusters describe private-cluster1 \
   --location=us-central1-f \
   --format="yaml(network, privateClusterConfig)"

```   

#### Check master-authorized-networks 

```

gcloud container clusters describe private-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --zone us-central1  

```

#### Add master-authorized-networks  IPS

```
gcloud container clusters update private-cluster1 \
    --zone us-central1-f \
    --enable-master-authorized-networks \
    --master-authorized-networks 172.22.4.10/32,172.22.2.20/32

gcloud container clusters describe private-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --zone us-central1-f      

```

### Create VM in Google Cloud with Custom Service Account

#### Service Account

```
 
gcloud iam service-accounts create SA_NAME \
    --description="DESCRIPTION" \
    --display-name="DISPLAY_NAME"

gcloud iam service-accounts create gke-sa \
    --description="GKE Service Account" \
    --display-name="GKE SA"

gcloud iam service-accounts list

gcloud iam service-accounts describe gke-sa@dev-project-786111.iam.gserviceaccount.com

```

#### Service Account membership

```

gcloud projects add-iam-policy-binding dev-project-786111 \
  --member="serviceAccount:gke-sa@dev-project-786111.iam.gserviceaccount.com" \
  --role="roles/monitoring.metricWriter"
  --condition None


gcloud projects add-iam-policy-binding dev-project-786111 \
  --member="serviceAccount:gke-sa@dev-project-786111.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"
  --condition None

gcloud projects add-iam-policy-binding dev-project-786111 \
  --member="serviceAccount:gke-sa@dev-project-786111.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser" \
  --condition None

gcloud projects add-iam-policy-binding dev-project-786111 \
  --member="serviceAccount:gke-sa@dev-project-786111.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.repoAdmin" \
  --condition None

  gcloud projects add-iam-policy-binding dev-project-786111 \
  --member="serviceAccount:gke-sa@dev-project-786111.iam.gserviceaccount.com" \
  --role="roles/container.clusterViewer" \
  --condition None

gcloud projects add-iam-policy-binding dev-project-786111 \
  --member="serviceAccount:gke-sa@dev-project-786111.iam.gserviceaccount.com" \
  --role="roles/container.admin" \
  --condition None

gcloud projects get-iam-policy dev-project-786111   \
--flatten="bindings[].members" \
--format='table(bindings.role)' \
--filter="bindings.members:gke-sa@dev-project-786111.iam.gserviceaccount.com"

```

#### Kubectl & gcloud-auth-Plugin Script

```
# Install Kubectl
!/bin/bash
sudo apt update -y
sudo apt upgrade -y
sudo apt autoclean -y
sudo apt install zip unzip wget net-tools vim nano htop tree telnet -y 

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl 

# Install gcloud-auth-Plugin

sudo apt-get update -y
sudo apt-get install apt-transport-https ca-certificates gnupg curl -y
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

sudo apt-get update && sudo apt-get install google-cloud-cli -y

grep -rhE ^deb /etc/apt/sources.list* | grep "cloud-sdk"


sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin -y

gke-gcloud-auth-plugin --version


# https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke
     

```

#### Kubecolor & Kubectl Alias

```

sudo wget https://github.com/hidetatz/kubecolor/releases/download/v0.0.25/kubecolor_0.0.25_Linux_x86_64.tar.gz
sudo tar zxv -f kubecolor_0.0.25_Linux_x86_64.tar.gz

echo "alias k=/home/aslam/" >> ~/.bashrc
echo "alias k=/home/aslam/kubecolor" >> ~/.bashrc

sudo rm -rf kubecolor_0.0.25_Linux_x86_64.tar.gz

 alias k=/home/aslam/
 alias k=/home/aslam/kubecolor     

```

#### Install Kubectx & Kubens

```
sudo snap install kubectx --classic

kubectx

```

#### Create K8s-VM

```
gcloud compute instances create k8s-client1 \
    --zone me-central1-a \
    --image-family ubuntu-2204-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-medium \
    --boot-disk-size 20GB \
    --boot-disk-type pd-balanced \
    --subnet k8s-vpc-sub6-me-central1 \
    --private-network-ip 172.22.4.10 \
    --service-account gke-sa@dev-project-786111.iam.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --metadata enable-oslogin=TRUE \
    --metadata-from-file startup-script=$HOME/web-scripts/kube-script.sh \
    --tags k8s-ssh-allow   

```

#### Connect GKE Cluster using GCP VM

```

gcloud container clusters list

gcloud container clusters get-credentials private-cluster1 --zone us-central1-f --project dev-project-786111 --internal-ip  

```

### Private Cluster in other region without Control plane global access 


```
gcloud container clusters create private-cluster2 \
    --zone us-west1-a \
    --node-locations us-west1-a,us-west1-b \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 172.22.4.10/32 \
    --private-endpoint-subnetwork k8s-vpc-master-sub2-us-west1 \
    --network k8s-vpc \
    --subnetwork  k8s-vpc-sub2-us-west1  \
    --cluster-secondary-range-name pod-cidr1 \
    --services-secondary-range-name service-cidr \
    --enable-private-nodes \
    --enable-private-endpoint \
    --enable-ip-alias \
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

#### Create K8s-VM on US-West1

```
gcloud compute instances create k8s-client2 \
    --zone us-west1-b \
    --image-family ubuntu-2204-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-medium \
    --boot-disk-size 50GB \
    --boot-disk-type pd-balanced \
    --subnet k8s-vpc-sub4-us-west1 \
    --private-network-ip 172.22.2.20 \
    --service-account gke-sa@dev-project-786111.iam.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --metadata enable-oslogin=TRUE \
    --metadata-from-file startup-script=$HOME/web-scripts/kube-script.sh \
    --tags k8s-ssh-allow 

```

#### Add master-authorized-networks  IPS

```
gcloud container clusters update private-cluster2 \
    --zone us-west1-a \
    --enable-master-authorized-networks \
    --master-authorized-networks 172.22.4.10/32,172.22.2.20/32

gcloud container clusters describe private-cluster2 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --zone us-west1-a     

```

#### Connect GKE Cluster using GCP VM from Same Region US-West1

```

gcloud container clusters list

gcloud container clusters get-credentials private-cluster2 --zone us-west1-a --project dev-project-786111 --internal-ip    

```

#### Deescribe Cluster Config File

```

gcloud container clusters list

gcloud container clusters describe private-cluster2 \
   --location=us-west1-a \
   --format="yaml(network, privateClusterConfig)"

``` 

 Note : without    --enable-master-global-access (Cluster only access from same region us-west1)
 Cluster wont access from any other region because of disable --enable-master-global-access

 masterGlobalAccessConfig: (thie setting is missing in privateClusterConfig)

### For Other Regions to Connect Private Cluster

 #### Update Private Cluster

 ```
 gcloud container clusters list

 gcloud container clusters update private-cluster2 \
    --location=us-west1-a \
    --enable-master-global-access 

gcloud container clusters describe private-cluster2 \
   --location=us-west1-a \
   --format="yaml(network, privateClusterConfig)"

``` 

Note : masterGlobalAccessConfig:
    enabled: true

 #### Connect GKE Cluster using GCP VM from Others Regions

```

gcloud container clusters list

gcloud container clusters get-credentials private-cluster2 --zone us-west1-a --project dev-project-786111 --internal-ip    

```   

### Private Cluster with custom master Cidr

```

gcloud container clusters create private-cluster3 \
    --region us-central1-c \
    --node-locations us-central1-a,us-central1-c \
    --release-channel "regular" \
    --cluster-version 1.30.5-gke.1443001 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 172.22.4.10/32 \
    --master-ipv4-cidr 172.16.2.0/28 \
    --network k8s-vpc \
    --subnetwork k8s-vpc-sub1-us-central1  \
    --cluster-secondary-range-name pod-cidr1 \
    --services-secondary-range-name service-cidr \
    --enable-private-nodes \
    --enable-private-endpoint \
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

gcloud container clusters delete private-cluster1 --zone us-central1-f
gcloud container clusters delete private-cluster2 --zone us-west1-a
gcloud container clusters delete private-cluster3 --zone us-central1-c

```


### Delete Client Vms

```
gcloud compute instances list

gcloud compute instances delete k8s-clien1 --zone me-central1-a 
gcloud compute instances delete k8s-client2 --zone us-west1-b

```

### Delete Service Account

```
gcloud iam service-accounts list

gcloud iam service-accounts delete gke-sa@dev-project-786111.iam.gserviceaccount.com

```
### NatGW for Private Nodes in GKE:

#### Cloud NAT for Region 1

```
gcloud compute addresses create natgw-gke-pip-us-central1  \
    --region us-central1

gcloud compute addresses list

gcloud compute addresses describe natgw-gke-pip-us-central1 --region us-central1

gcloud compute routers create gke-nat-router-us-central1 \
    --network k8s-vpc \
    --region us-central1

gcloud compute routers list

Note : For All Subnet

gcloud compute routers nats create gke-natgw-us-central1 \
    --router gke-nat-router-us-central1 \
    --region us-central1 \
    --nat-external-ip-pool natgw-gke-pip-us-central1 \
    --nat-all-subnet-ip-ranges \
    --min-ports-per-vm 128 \
    --max-ports-per-vm 512 \
    --enable-logging

Note : For only one Subnet

gcloud compute routers nats create gke-natgw-us-central1 \
    --router gke-nat-router-us-central1 \
    --region us-central1 \
    --nat-external-ip-pool natgw-gke-pip-us-central1 \
    --nat-custom-subnet-ip-ranges k8s-vpc-subnet-us-central1 \
    --min-ports-per-vm 128 \
    --max-ports-per-vm 512 \
    --enable-logging


gcloud compute routers nats list --router gke-nat-router-us-central1 --region us-central1
gcloud compute routers nats describe gke-natgw-us-central1 --router gke-nat-router-us-central1 --region us-central1

```

#### Cloud NAT for Region 2

```
gcloud compute addresses create natgw-gke-pip-us-west1  \
    --region us-west1

gcloud compute addresses list

gcloud compute addresses describe natgw-gke-pip-us-west1 --region us-west1

gcloud compute routers create gke-nat-router-us-west1 \
    --network k8s-vpc \
    --region us-west1

gcloud compute routers list

Note : For All Subnet

gcloud compute routers nats create gke-natgw-us-west1 \
    --router gke-nat-router-us-west1 \
    --region us-west1 \
    --nat-external-ip-pool natgw-gke-pip-us-west1 \
    --nat-all-subnet-ip-ranges \
    --min-ports-per-vm 128 \
    --max-ports-per-vm 512 \
    --enable-logging


gcloud compute routers nats list --router gke-nat-router-us-west1 --region us-west1
gcloud compute routers nats describe gke-natgw-us-west1 --router gke-nat-router-us-west1 --region us-west1

```


