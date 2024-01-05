# rilb-multi-region-dns-test-01
testing using Cloud DNS to provide failover for multiple services on GKE across multiple regions. this demo uses the `default` VPC, which is not recommended for production.

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
kubectl --context=$KUBECTX_1 create namespace gateway
kubectl --context=$KUBECTX_2 create namespace gateway

kubectl --context=$KUBECTX_1 create namespace service-a
kubectl --context=$KUBECTX_2 create namespace service-a

kubectl --context=$KUBECTX_1 create namespace service-b
kubectl --context=$KUBECTX_2 create namespace service-b

kubectl --context=$KUBECTX_1 apply -k whereami/service-a
kubectl --context=$KUBECTX_2 apply -k whereami/service-a

kubectl --context=$KUBECTX_1 apply -k whereami/service-b
kubectl --context=$KUBECTX_2 apply -k whereami/service-b
```

### set up load balancers

```
kubectl --context=$KUBECTX_1 apply -f gateway/ # create gateway resource and enable global access policy
kubectl --context=$KUBECTX_2 apply -f gateway/ # create gateway resource and enable global access policy
```

### create HTTPRoutes for service-a

```
kubectl --context=$KUBECTX_1 apply -f httproute/service-a-httproute.yaml
kubectl --context=$KUBECTX_2 apply -f httproute/service-a-httproute.yaml

# testing for my local project VMs
curl -v --header "Host: service-a.internal.example.com" http://10.128.0.24 # region_1
curl -v --header "Host: service-a.internal.example.com" http://10.150.0.25 # region_2
```

### create HTTPRoutes for service-b

```
kubectl --context=$KUBECTX_1 apply -f httproute/service-b-httproute.yaml
kubectl --context=$KUBECTX_2 apply -f httproute/service-b-httproute.yaml

# testing for my local project VMs
curl -v --header "Host: service-b.internal.example.com" http://10.128.0.24 # region_1
curl -v --header "Host: service-b.internal.example.com" http://10.150.0.25 # region_2
```

### create DNS zone

```
gcloud dns --project=$PROJECT managed-zones create geotest --description="" --dns-name="internal.example.com." --visibility="private" --networks="https://www.googleapis.com/compute/v1/projects/${PROJECT}/global/networks/${VPC}"
```

### create geolocated DNS records

```
gcloud dns --project=$PROJECT record-sets create service-a.internal.example.com. --zone="geotest" --type="A" --ttl="5" --routing-policy-type="GEO" --enable-health-checking --routing-policy-data="${REGION_1}=projects/${PROJECT}/regions/${REGION_1}/forwardingRules/gkegw1-s67w-gateway-internal-http-v276ajbysrz2;${REGION_2}=projects/${PROJECT}/regions/${REGION_2}/forwardingRules/gkegw1-owtg-gateway-internal-http-n2zw5w650pmn" # replace forwarding rules with your own

gcloud dns --project=$PROJECT record-sets create service-b.internal.example.com. --zone="geotest" --type="A" --ttl="5" --routing-policy-type="GEO" --enable-health-checking --routing-policy-data="${REGION_1}=projects/${PROJECT}/regions/${REGION_1}/forwardingRules/gkegw1-s67w-gateway-internal-http-v276ajbysrz2;${REGION_2}=projects/${PROJECT}/regions/${REGION_2}/forwardingRules/gkegw1-owtg-gateway-internal-http-n2zw5w650pmn" # replace forwarding rules with your own
```