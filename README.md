# envoy-edge-proxy-testing
Comparison of different Envoy proxy implementations on GKE

### deploy clusters

using public nodes to simplify grabbing external images

> note: using GKE Standard instead of GKE Autopilot because Envoy Gateway installation puked when using Autopilot

> note 2: generating these gcloud commands via Pantheon still doesn't include the Gateway API enablement selection once it's been selected
```
export PROJECT=spanner-reporter-01 # replace with your own project

# create GKE Standard cluster for Envoy Gateway
gcloud beta container --project $PROJECT clusters create "gke-envoy-gateway-std" --async --region "us-central1" --no-enable-basic-auth --cluster-version "1.30.1-gke.1329003" --release-channel "rapid" --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM,STORAGE,POD,DEPLOYMENT,STATEFULSET,DAEMONSET,HPA,CADVISOR,KUBELET --enable-ip-alias --network "projects/${PROJECT}/global/networks/default" --subnetwork "projects/${PROJECT}/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "32" --enable-autoscaling --min-nodes "0" --max-nodes "2" --location-policy "BALANCED" --security-posture=standard --workload-vulnerability-scanning=disabled --enable-dataplane-v2 --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --binauthz-evaluation-mode=DISABLED --enable-managed-prometheus --enable-shielded-nodes --gateway-api=standard

# create GKE Standard cluster for GCLB (managed ALB)
gcloud beta container --project $PROJECT clusters create "gke-managed-alb-std" --async --region "us-central1" --no-enable-basic-auth --cluster-version "1.30.1-gke.1329003" --release-channel "rapid" --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM,STORAGE,POD,DEPLOYMENT,STATEFULSET,DAEMONSET,HPA,CADVISOR,KUBELET --enable-ip-alias --network "projects/${PROJECT}/global/networks/default" --subnetwork "projects/${PROJECT}/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "32" --enable-autoscaling --min-nodes "0" --max-nodes "2" --location-policy "BALANCED" --security-posture=standard --workload-vulnerability-scanning=disabled --enable-dataplane-v2 --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --binauthz-evaluation-mode=DISABLED --enable-managed-prometheus --enable-shielded-nodes --gateway-api=standard

# check cluster creation status (waiting for clusters to be ready)
gcloud container clusters list

# you're ready to go when things look like this (i.e. STATUS is RUNNING)
$ gcloud container clusters list
NAME                   LOCATION     MASTER_VERSION      MASTER_IP      MACHINE_TYPE  NODE_VERSION        NUM_NODES  STATUS
gke-envoy-gateway-std  us-central1  1.30.1-gke.1329003  35.222.70.64   e2-medium     1.30.1-gke.1329003  3          RUNNING
gke-managed-alb-std    us-central1  1.30.1-gke.1329003  34.134.28.108  e2-medium     1.30.1-gke.1329003  3          RUNNING

# once clusters are ready, grab credentials to kubeconfig
gcloud container clusters get-credentials gke-envoy-gateway-std --region us-central1
gcloud container clusters get-credentials gke-managed-alb-std --region us-central1

# rename contexts for ease of use
kubectl config rename-context gke_${PROJECT}_us-central1_gke-envoy-gateway-std gke-envoy-gateway-std
kubectl config rename-context gke_${PROJECT}_us-central1_gke-managed-alb-std gke-managed-alb-std
```

### deploy sample app 

```
kubectl --context=gke-envoy-gateway-std create ns frontend
kubectl --context=gke-managed-alb-std create ns frontend

kubectl --context=gke-envoy-gateway-std apply -k whereami-frontend/variant
kubectl --context=gke-managed-alb-std apply -k whereami-frontend/variant
```

### set up Envoy Gateway

using https://gateway.envoyproxy.io/docs/install/install-yaml/

```
kubectl --context=gke-envoy-gateway-std apply --server-side -f https://github.com/envoyproxy/gateway/releases/download/v1.1.0/install.yaml

```