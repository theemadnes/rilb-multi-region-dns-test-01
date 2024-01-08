# rilb-multi-region-dns-test-01
testing using Cloud DNS to provide failover for multiple services on GKE across multiple regions. this demo uses the `default` VPC, which is not recommended for production.

### setup 

```
export PROJECT=e2m-private-test-01
export VPC=default
export REGION_1=us-central1
export REGION_2=us-east4
export CLUSTER_1=edge-to-mesh-01
export CLUSTER_2=edge-to-mesh-02
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

# testing for my local project VMs (replace ILB VIPs with your own)
curl -v --header "Host: service-a.internal.example.com" http://10.128.0.24 # region_1
curl -v --header "Host: service-a.internal.example.com" http://10.150.0.25 # region_2
```

### create HTTPRoutes for service-b

```
kubectl --context=$KUBECTX_1 apply -f httproute/service-b-httproute.yaml
kubectl --context=$KUBECTX_2 apply -f httproute/service-b-httproute.yaml

# testing for my local project VMs (replace ILB VIPs with your own)
curl -v --header "Host: service-b.internal.example.com" http://10.128.0.24 # region_1
curl -v --header "Host: service-b.internal.example.com" http://10.150.0.25 # region_2
```

### create DNS zone

```
gcloud dns --project=$PROJECT managed-zones create geotest --description="" --dns-name="internal.example.com." --visibility="private" --networks="https://www.googleapis.com/compute/v1/projects/${PROJECT}/global/networks/${VPC}"
```

### create geolocated DNS records

```
# NOTE: replace forwarding rule names with your own
gcloud dns --project=$PROJECT record-sets create service-a.internal.example.com. --zone="geotest" --type="A" --ttl="5" --routing-policy-type="GEO" --enable-health-checking --routing-policy-data="${REGION_1}=projects/${PROJECT}/regions/${REGION_1}/forwardingRules/gkegw1-s67w-gateway-internal-http-v276ajbysrz2;${REGION_2}=projects/${PROJECT}/regions/${REGION_2}/forwardingRules/gkegw1-owtg-gateway-internal-http-n2zw5w650pmn"

# NOTE: replace forwarding rule names with your own
gcloud dns --project=$PROJECT record-sets create service-b.internal.example.com. --zone="geotest" --type="A" --ttl="5" --routing-policy-type="GEO" --enable-health-checking --routing-policy-data="${REGION_1}=projects/${PROJECT}/regions/${REGION_1}/forwardingRules/gkegw1-s67w-gateway-internal-http-v276ajbysrz2;${REGION_2}=projects/${PROJECT}/regions/${REGION_2}/forwardingRules/gkegw1-owtg-gateway-internal-http-n2zw5w650pmn"

# testing for my local project VMs (check from VMs in each region)
curl http://service-a.internal.example.com
curl http://service-b.internal.example.com
```

### test failover

```
# testing from region_1, so killing service-a pods in region_1
kubectl --context=$KUBECTX_1 -n service-a scale --replicas=0 deployment/whereami-service-a

$ curl http://service-a.internal.example.com -v 
*   Trying 10.128.0.24:80...
* Connected to service-a.internal.example.com (10.128.0.24) port 80 (#0)
> GET / HTTP/1.1
> Host: service-a.internal.example.com
> User-Agent: curl/7.74.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 503 Service Unavailable
< content-length: 19
< content-type: text/plain
< date: Fri, 05 Jan 2024 02:48:23 GMT
< via: 1.1 google
< 
* Connection #0 to host service-a.internal.example.com left intact

# hmmm not failing over....
```

### testing cross region internal ALB

attempting service-level failover using [x-region ALB](https://cloud.google.com/load-balancing/docs/l7-internal#cross-region-failover), following steps documented [here](https://cloud.google.com/load-balancing/docs/l7-internal/setting-up-l7-cross-reg-hybrid) (but using traditional IP-port standalone [NEGs](https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg))

```
# create proxy subnets for x-region ALB
gcloud compute networks subnets create $REGION_1-x-pos \
    --purpose=GLOBAL_MANAGED_PROXY \
    --role=ACTIVE \
    --region=$REGION_1 \
    --network=$VPC \
    --range=172.16.30.0/24

gcloud compute networks subnets create $REGION_2-x-pos \
    --purpose=GLOBAL_MANAGED_PROXY \
    --role=ACTIVE \
    --region=$REGION_2 \
    --network=$VPC \
    --range=172.16.40.0/24

# create FW rules for x-region ALB
gcloud compute firewall-rules create fw-allow-health-check \
    --network=$VPC \
    --action=allow \
    --direction=ingress \
    --target-tags=allow-health-check \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --rules=tcp:8080

gcloud resource-manager tags keys create allow-proxy-only-subnet \
    --parent=projects/$PROJECT \
    --purpose=GCE_FIREWALL \
    --purpose-data=network=$PROJECT/$VPC

gcloud container clusters update $CLUSTER_1 \
    --autoprovisioning-network-tags="allow-proxy-only-subnet","allow-health-check" \
    --region $REGION_1

gcloud container clusters update $CLUSTER_2 \
    --autoprovisioning-network-tags="allow-proxy-only-subnet","allow-health-check" \
    --region $REGION_2

gcloud compute firewall-rules create fw-allow-proxy-only-subnet \
    --network=$VPC \
    --action=allow \
    --direction=ingress \
    --target-tags=allow-proxy-only-subnet \
    --source-ranges=172.16.30.0/24,172.16.40.0/24 \
    --rules=tcp:8080

# create HTTP health check for x-region iALB
gcloud compute health-checks create http gil7-basic-check \
   --use-serving-port \
   --global

# (optional but recommmended: if using GKE autopilot mode:
# follow issuetracker.google.com/228379727 to 'pre-warm' Aupilot clusters to
# use all available zones in region so NEGs are evenly distributed)

# create services 
kubectl --context=$KUBECTX_1 apply -f service-negs/service-a-neg.yaml
kubectl --context=$KUBECTX_2 apply -f service-negs/service-a-neg.yaml

kubectl --context=$KUBECTX_1 apply -f service-negs/service-b-neg.yaml
kubectl --context=$KUBECTX_2 apply -f service-negs/service-b-neg.yaml

# create LB backend services
gcloud compute backend-services create service-a \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --protocol=HTTP \
  --enable-logging \
  --logging-sample-rate=1.0 \
  --health-checks=gil7-basic-check \
  --global-health-checks \
  --global

gcloud compute backend-services create service-b \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --protocol=HTTP \
  --enable-logging \
  --logging-sample-rate=1.0 \
  --health-checks=gil7-basic-check \
  --global-health-checks \
  --global

# add backends

### hack-y way to get the negs added to the backend services
for ZONE in $REGION_1-a $REGION_1-b $REGION_1-c $REGION_1-d $REGION_1-e $REGION_1-f $REGION_2-a $REGION_2-b $REGION_2-c $REGION_2-d $REGION_2-e $REGION_2-f
do
	gcloud compute backend-services add-backend service-a \
    --global \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=10000 \
    --network-endpoint-group=whereami-service-a-neg \
    --network-endpoint-group-zone=$ZONE
done

for ZONE in $REGION_1-a $REGION_1-b $REGION_1-c $REGION_1-d $REGION_1-e $REGION_1-f $REGION_2-a $REGION_2-b $REGION_2-c $REGION_2-d $REGION_2-e $REGION_2-f
do
	gcloud compute backend-services add-backend service-b \
    --global \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=10000 \
    --network-endpoint-group=whereami-service-b-neg \
    --network-endpoint-group-zone=$ZONE
done

# create URL map
gcloud compute url-maps create gil7-map \
  --default-service=service-a \
  --global

gcloud compute url-maps add-path-matcher gil7-map --path-matcher-name=service-a --default-service=service-a
gcloud compute url-maps add-host-rule gil7-map --hosts='service-a.internal.example.com' --path-matcher-name=service-a
gcloud compute url-maps add-path-matcher gil7-map --path-matcher-name=service-b --default-service=service-b --new-hosts='service-b.internal.example.com'

# create target proxy
gcloud compute target-http-proxies create gil7-http-proxy \
  --url-map=gil7-map \
  --global

# create forwarding rule

gcloud compute forwarding-rules create gil7-forwarding-rule-$REGION_1 \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --network=$VPC \
  --subnet=default \
  --subnet-region=$REGION_1 \
  --ports=80 \
  --target-http-proxy=gil7-http-proxy \
  --global

gcloud compute forwarding-rules create gil7-forwarding-rule-$REGION_2 \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --network=$VPC \
  --subnet=default \
  --subnet-region=$REGION_2 \
  --ports=80 \
  --target-http-proxy=gil7-http-proxy \
  --global

# testing failure
kubectl --context=$KUBECTX_1 -n service-a scale --replicas=0 deployment/whereami-service-a
kubectl --context=$KUBECTX_1 -n service-b scale --replicas=0 deployment/whereami-service-b

# testing after failover 
#curl -v --header "Host: service-a.internal.example.com" http://$(gcloud compute forwarding-rules describe gil7-forwarding-rule-$REGION_1 --global --format json | jq ".IPAddress" | tr -d '"')
curl -v --header "Host: service-a.internal.example.com" http://10.128.0.38
#curl -v --header "Host: service-b.internal.example.com" http://$(gcloud compute forwarding-rules describe gil7-forwarding-rule-$REGION_1 --global --format json | jq ".IPAddress" | tr -d '"')
curl -v --header "Host: service-a.internal.example.com" http://10.150.0.39

# restore replicas
kubectl --context=$KUBECTX_1 -n service-a scale --replicas=3 deployment/whereami-service-a
kubectl --context=$KUBECTX_2 -n service-a scale --replicas=3 deployment/whereami-service-a
kubectl --context=$KUBECTX_1 -n service-b scale --replicas=3 deployment/whereami-service-b
kubectl --context=$KUBECTX_2 -n service-b scale --replicas=3 deployment/whereami-service-b
```

### scratch / ignore

```
# test health check for service-a
kubectl --context=$KUBECTX_1 apply -f health-check/service-a-health-check.yaml
kubectl --context=$KUBECTX_2 apply -f health-check/service-a-health-check.yaml

# test health check for service-b
kubectl --context=$KUBECTX_1 apply -f health-check/service-b-health-check.yaml
kubectl --context=$KUBECTX_2 apply -f health-check/service-b-health-check.yaml
```