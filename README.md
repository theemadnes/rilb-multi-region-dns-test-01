# rilb-multi-region-dns-test-01
testing using Cloud DNS to provide failover for multiple services on GKE across multiple regions 

### setup 

```
export PROJECT=e2m-private-test-01
export VPC=default
export REGION_1=us-central1
export REGION_2=us-east4
export KUBECTX_1=gke_e2m-private-test-01_us-central1_edge-to-mesh-01
export KUBECTX_2=gke_e2m-private-test-01_us-east4_edge-to-mesh-02
gcloud config set project $PROJECT
```

### create proxy subnets

```
gcloud compute networks subnets create $REGION_1-pos \
    --purpose=REGIONAL_MANAGED_PROXY \
    --role=ACTIVE \
    --region=$REGION_1 \
    --network=$VPC \
    --range=172.16.10.0/24

gcloud compute networks subnets create $REGION_2-pos \
    --purpose=REGIONAL_MANAGED_PROXY \
    --role=ACTIVE \
    --region=$REGION_2 \
    --network=$VPC \
    --range=172.16.20.0/24
```

### deploy services

```
kubectl --context=$KUBECTX_1 create namespace service-a
kubectl --context=$KUBECTX_2 create namespace service-a

kubectl --context=$KUBECTX_1 create namespace service-b
kubectl --context=$KUBECTX_2 create namespace service-b

kubectl --context=$KUBECTX_1 apply -k whereami/service-a
kubectl --context=$KUBECTX_2 apply -k whereami/service-a

kubectl --context=$KUBECTX_1 apply -k whereami/service-b
kubectl --context=$KUBECTX_2 apply -k whereami/service-b

```