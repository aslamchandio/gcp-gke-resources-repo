# GCP VPC For Google Kubernetes Engine


## What this does?

This document describes how to create, modify, and delete Virtual Private Cloud (VPC) networks and subnetworks. Before reading this document, ensure that you're familiar with the characteristics of VPC networks as described in VPC networks. Networks and subnets are different resources in Google Cloud.

## Create networks
You can choose to create an auto mode or custom mode VPC network. Each new network that you create must have a unique name within the same project.

## Create an auto mode VPC network
When you create an auto mode VPC network, one subnet is created in each Google Cloud region. As new regions become available, new subnets in those regions are automatically added to the auto mode VPC network. IPv4 ranges for the automatically created subnets come from a predetermined set of ranges. All auto mode VPC networks use the same set of IPv4 ranges.

Subnets with IPv6 ranges are not supported on auto mode VPC networks. Create a custom mode VPC network if you want to create dual-stack subnets.

## Create a custom mode VPC network with only IPv4 subnets

For custom mode VPC networks, create a network, then create the subnets that you want within a region. You do not have to specify subnets for all regions right away, or even at all, but you cannot create instances in a region that has no subnet defined. Finally, define the firewall rules for your network.

To create a custom mode VPC network with only IPv4 subnets, follow these steps.

```
gcloud config set accessibility/screen_reader false
gcloud config set compute/region us-central
gcloud config set compute/zone us-central1-c

```

```
gcloud compute networks create NETWORK \
    --subnet-mode=custom \
    --bgp-routing-mode=DYNAMIC_ROUTING_MODE \
    --mtu=MTU

gcloud compute networks create k8s-vpc \
    --subnet-mode custom \
    --bgp-routing-mode regional \
    --mtu 1460  

# note : or --bgp-routing-mode global

gcloud compute networks list  

```

. NETWORK: a name for the VPC network.
. DYNAMIC_ROUTING_MODE: controls the behavior of Cloud Routers in the   network. Can be either global or regional. The default is regional. For more information, see dynamic routing mode.
. MTU: the maximum transmission unit (MTU), which is the largest packet size of the network. MTU can be set to any value from 1300 to 8896. The default is 1460. Before setting the MTU to a value higher than 1460, review Maximum transmission unit.

Next, add subnets to your network.

Create a custom mode VPC network with a IPv4 subnet

```
gcloud compute networks subnets create k8s-vpc-sub1-us-central1 \
  --network k8s-vpc \
  --range 192.168.16.0/20 \
  --region us-central1 \
  --enable-flow-logs \
  --enable-private-ip-google-access

gcloud compute networks subnets list --network k8s-vpc
gcloud compute networks subnets list --filter  network:k8s-vpc  

```

### Add Pod & Service CIDR in Subnet

### Note: For Pod CIDR(10.244.0.0/16,10.245.0.0/16)  For Service CIDR 10.32.0.0/16

```
gcloud compute networks subnets update k8s-vpc-sub1-us-central1 \
  --region us-central1 \
  --add-secondary-ranges=pod-cidr1=10.244.0.0/16

gcloud compute networks subnets update k8s-vpc-sub1-us-central1 \
  --region us-central1 \
  --add-secondary-ranges=pod-cidr2=10.245.0.0/16 


gcloud compute networks subnets update k8s-vpc-sub1-us-central1 \
  --region us-central1 \
  --add-secondary-ranges=service-cidr=10.32.0.0/16

gcloud compute networks subnets describe k8s-vpc-sub1-us-central1 

gcloud compute networks subnets list --network k8s-vpc
gcloud compute networks subnets list --filter  network:k8s-vpc

```

Note : You can also Configure subnet with Pod and service CIDR in one command

```
gcloud compute networks subnets create k8s-vpc-subnet-us-central1 \
    --network k8s-vpc \
    --range 192.168.16.0/20 \
    --region us-central1 \    
    --secondary-range pod-cidr1=10.244.0.0/16,pod-cidr2=10.245.0.0/16,service-cidr=10.32.0.0/16 \
    --enable-flow-logs \
    --enable-private-ip-google-access
    
gcloud compute routes list \
    --filter="default-internet-gateway k8s-vpc"

```

###  Subnets For Masters Node

```
gcloud compute networks subnets create k8s-vpc-master-sub1-us-central1 \
  --network k8s-vpc \
  --range 172.16.1.0/28 \
  --region us-central1 \
  --enable-flow-logs 

gcloud compute networks subnets list --network k8s-vpc
gcloud compute networks subnets list --filter  network:k8s-vpc

```

###  Subnet For Regional Load Balancer(Gateway API)

```
gcloud compute networks subnets create k8s-vpc-proxy-only-subnet1 \
    --network k8s-vpc \
    --range 192.168.1.0/24 \
    --region us-central1 \
    --purpose REGIONAL_MANAGED_PROXY \
    --role ACTIVE

gcloud compute networks subnets list --filter  network:k8s-vpc

```



## Other Region Subnet for gke

```
gcloud compute networks subnets create k8s-vpc-sub2-us-west1 \
  --network k8s-vpc \
  --range 192.168.32.0/20 \
  --region us-west1 \
  --enable-flow-logs \
  --enable-private-ip-google-access

```

### Add Pod & Service CIDR in Subnet

### # Note: For Pod CIDR(10.242.0.0/16,10.243.0.0/16)  For Service CIDR 10.33.0.0/16

```
gcloud compute networks subnets update k8s-vpc-sub2-us-west1 \
  --region us-west1 \
  --add-secondary-ranges=pod-cidr1=10.241.0.0/16

gcloud compute networks subnets update k8s-vpc-sub2-us-west1 \
  --region us-west1 \
  --add-secondary-ranges=pod-cidr2=10.242.0.0/16 


gcloud compute networks subnets update k8s-vpc-sub2-us-west1 \
  --region us-west1 \
  --add-secondary-ranges=service-cidr=10.33.0.0/16

gcloud compute networks subnets describe k8s-vpc-sub2-us-west1  --region us-west1


gcloud compute networks subnets list --network k8s-vpc
gcloud compute networks subnets list --filter  network:k8s-vpc

```

Note : You can also Configure subnet with Pod and service CIDR in one command

```
gcloud compute networks subnets create k8s-vpc-sub2-us-west1\
    --network k8s-vpc \
    --range 192.168.16.0/20 \
    --region us-west1 \    
    --secondary-range pod-cidr1=10.241.0.0/16,pod-cidr2=10.242.0.0/16,service-cidr=10.33.0.0/16 \
    --enable-flow-logs \
    --enable-private-ip-google-access
    
gcloud compute routes list \
    --filter="default-internet-gateway k8s-vpc"

```


###  Subnets For Masters Node

```
gcloud compute networks subnets create k8s-vpc-master-sub2-us-west1 \
  --network k8s-vpc \
  --range 172.16.1.16/28 \
  --region us-west1 \
  --enable-flow-logs

gcloud compute networks subnets list --network k8s-vpc
gcloud compute networks subnets list --filter  network:k8s-vpc

```

###  Subnet For Regional Load Balancer(Gateway API)

```
gcloud compute networks subnets create k8s-vpc-proxy-only-subnet2 \
    --network k8s-vpc \
    --range 192.168.2.0/24  \
    --region us-west1 \
    --purpose REGIONAL_MANAGED_PROXY \
    --role ACTIVE 

gcloud compute networks subnets list --filter  network:k8s-vpc

```

##  Other Subnets

```
gcloud compute networks subnets create k8s-vpc-sub3-us-central1 \
  --network k8s-vpc \
  --range 172.22.1.0/24 \
  --region us-central1 \
  --enable-flow-logs \
  --enable-private-ip-google-access

gcloud compute networks subnets create k8s-vpc-sub4-us-west1 \
  --network k8s-vpc \
  --range 172.22.2.0/24 \
  --region us-west1 \
  --enable-flow-logs \
  --enable-private-ip-google-access
 

gcloud compute networks subnets create k8s-vpc-sub5-euro-west2 \
  --network k8s-vpc \
  --range 172.22.3.0/24 \
  --region europe-west2 \
  --enable-flow-logs \
  --enable-private-ip-google-access 

gcloud compute networks subnets create k8s-vpc-sub6-me-central1 \
  --network k8s-vpc \
  --range 172.22.4.0/24 \
  --region me-central1 \
  --enable-flow-logs \
  --enable-private-ip-google-access 


```