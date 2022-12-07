
```bash

export CTX_CLUSTER1=kind-kind-1
export CTX_CLUSTER2=kind-kind-2

cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
  meshConfig:
    accessLogFile: /dev/stdout
EOF

istioctl install --set values.pilot.env.EXTERNAL_ISTIOD=true --context="${CTX_CLUSTER1}" -f cluster1.yaml -y

curl -O https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/gen-eastwest-gateway.sh
chmod +x gen-eastwest-gateway.sh

./gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster1 --network network1 | \
    istioctl --context="${CTX_CLUSTER1}" install -y -f -

kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system

kubectl apply --context="${CTX_CLUSTER1}" -n istio-system -f expose-istiod.yaml

kubectl --context="${CTX_CLUSTER2}" create namespace istio-system
kubectl --context="${CTX_CLUSTER2}" annotate namespace istio-system topology.istio.io/controlPlaneClusters=cluster1

export DISCOVERY_ADDRESS=$(kubectl \
    --context="${CTX_CLUSTER1}" \
    -n istio-system get svc istio-eastwestgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: remote
  values:
    istiodRemote:
      injectionPath: /inject/cluster/cluster2/net/network1
    global:
      remotePilotAddress: ${DISCOVERY_ADDRESS}
EOF

istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml -y

istioctl x create-remote-secret \
    --context="${CTX_CLUSTER2}" --server 172.18.0.4 \
    --name=cluster2 | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"

k2 apply -f ubuntu-with-istio.yaml

```

warn: server in Kubeconfig is https://127.0.0.1:62156. This is likely not reachable from inside the cluster, if you're using Kubernetes in Docker, pass --server with the container IP for the API Server