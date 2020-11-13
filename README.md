# Istio GRPC Load balancing Issue on multi-cluster

# TLDR

The traffic in multi-cluster should be consider under the same mesh.

So the virtual Service should use gateway `mesh`.

# Prepare

Create Google Cloud Platform project and enable APIs.

Install [gcloud SDK](https://cloud.google.com/sdk/docs/install).


Create GKE clusters.

```
# Cluster - client
gcloud container clusters create cluster-1 \
    --machine-type e2-standard-2 \
    --zone asia-east1-a \
    --enable-ip-alias

# Cluster - server
gcloud container clusters create cluster-2 \
    --machine-type e2-standard-2 \
    --zone us-west2-a \
    --enable-ip-alias
```

Export environment variables

```
export GOOGLE_CLOUD_PROJECT=<YOUR_PROJECT_NAME>
export CTX_CLUSTER1="gke_$GOOGLE_CLOUD_PROJECT_asia-east1-a_cluster-1"
export CTX_CLUSTER2="gke_$GOOGLE_CLOUD_PROJECT_us-west2-a_cluster-2"
```

Download Istio 1.6.8

```
export ISTIO_VERSION=1.6.8
export ISTIO_PATH=$PWD/istio-$ISTIO_VERSION
curl -L https://istio.io/downloadIstio | sh -
export PATH=$ISTIO_PATH/bin:$PATH
```

# GRPC Image

```
git clone https://github.com/GoogleCloudPlatform/istio-samples.git

cd istio-samples/sample-apps/grpc-greeter-go
gcloud builds submit server -t gcr.io/$GOOGLE_CLOUD_PROJECT/grpc-greeter-go-server
cd -

cd istio-samples/sample-apps/grpc-greeter-go/client
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o grpc-client -installsuffix "static" .
cd -
```

# Setup Cluster 2

Context switch

```
kubectl config use-context $CTX_CLUSTER2
```

Install Itsio

```
cd $ISTIO_PATH

kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
    --from-file=samples/certs/ca-cert.pem \
    --from-file=samples/certs/ca-key.pem \
    --from-file=samples/certs/root-cert.pem \
    --from-file=samples/certs/cert-chain.pem

istioctl install \
    -f manifests/examples/multicluster/values-istio-multicluster-gateways.yaml \
    --set meshConfig.accessLogFile=/dev/stdout

cd -
```

Setup Cluster 2

```
kubectl label namespace default istio-injection=enabled
kubectl apply -f cluster-2
```

```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: greeter
spec:
  replicas: 3
  selector:
    matchLabels:
      app: greeter
  template:
    metadata:
      labels:
        app: greeter
    spec:
      containers:
      - args:
        - --address=127.0.0.1:8080
        image: gcr.io/$GOOGLE_CLOUD_PROJECT/grpc-greeter-go-server
        imagePullPolicy: Always
        livenessProbe:
          exec:
            command:
            - /bin/grpc_health_probe
            - -addr=:8080
          initialDelaySeconds: 2
        name: greeter
        ports:
        - containerPort: 8080
        readinessProbe:
          exec:
            command:
            - /bin/grpc_health_probe
            - -addr=:8080
          initialDelaySeconds: 2
EOF
```

Wait service ready

```
export CLUSTER2_GW_ADDR=$(kubectl get svc \
    --selector=app=istio-internal-ingressgateway \
    -n istio-system \
    -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
```

GRPC Client

```
kubectl cp istio-samples/sample-apps/grpc-greeter-go/client/grpc-client sleep:/grpc-client -c sleep
```

Test

```
kubectl exec -it sleep -- /grpc-client --address=greeter:8080 --insecure=true
```

Output (balance)

```
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-v76cq
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-v76cq
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-qdd7v
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-v76cq
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-v76cq
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-v76cq
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-qdd7v
2020/11/13 08:16:20 Hello world from greeter-68f4cd6c6-v76cq
```

# Setup Cluster 1

Context switch

```
kubectl config use-context $CTX_CLUSTER1
```

Install Itsio

```
cd $ISTIO_PATH

kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
    --from-file=samples/certs/ca-cert.pem \
    --from-file=samples/certs/ca-key.pem \
    --from-file=samples/certs/root-cert.pem \
    --from-file=samples/certs/cert-chain.pem

istioctl install \
    -f manifests/examples/multicluster/values-istio-multicluster-gateways.yaml \
    --set meshConfig.accessLogFile=/dev/stdout

cd -
```

Setup Cluster 1

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-dns
  namespace: kube-system
data:
  stubDomains: |
    {
      "global": ["$(kubectl get svc -n istio-system istiocoredns -o jsonpath={.spec.clusterIP})"],
      "cluster-2": ["$(kubectl get svc -n istio-system istiocoredns -o jsonpath={.spec.clusterIP})"]
    }
EOF
```

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: greeter-cluster-2
spec:
  hosts:
  - greeter.default.cluster-2
  location: MESH_INTERNAL
  ports:
  - name: grpc
    number: 8080
    protocol: GRPC
  resolution: DNS
  addresses:
  - 240.0.0.2
  endpoints:
  - address: ${CLUSTER2_GW_ADDR}
    ports:
      grpc: 15443
EOF
```

```
kubectl label namespace default istio-injection=enabled
kubectl apply -f cluster-1
```

GRPC Client

```
kubectl cp istio-samples/sample-apps/grpc-greeter-go/client/grpc-client sleep:/grpc-client -c sleep
```

Test

```
kubectl exec -it sleep -- /grpc-client --address=greeter.default.cluster-2:8080 --insecure=true
```

Output (Not balance)

```
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:16:42 Hello world from greeter-68f4cd6c6-z5dr4
```

# Resolution

Uncomment `cluster-2/virtualservice.yaml`: `#- mesh`

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: greeter
spec:
  gateways:
  - istio-internal-ingressgateway
  - mesh
  hosts:
  - 'greeter'
  - 'greeter.default.cluster-2'
  - 'greeter.default.svc.cluster.local'
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: greeter
        port:
          number: 8080
```

Apply change

```
kubectl config use-context $CTX_CLUSTER2
kubectl apply -f cluster-2
```

Test

```
kubectl config use-context $CTX_CLUSTER1
kubectl exec -it sleep -- /grpc-client --address=greeter.default.cluster-2:8080 --insecure=true
```

Output (balance)

```
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-qdd7v
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-qdd7v
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-qdd7v
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-z5dr4
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-v76cq
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-qdd7v
2020/11/13 08:17:07 Hello world from greeter-68f4cd6c6-z5dr4
```
