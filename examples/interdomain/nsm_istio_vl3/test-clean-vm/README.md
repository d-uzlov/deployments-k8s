

```bash
cat <<EOF > ./vm-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "${CLUSTER}"
      network: "${CLUSTER_NETWORK}"
EOF
```

istioctl install -f vm-cluster.yaml --kubeconfig=$KUBECONFIG1

curl -O https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/gen-eastwest-gateway.sh
chmod +x gen-eastwest-gateway.sh
./gen-eastwest-gateway.sh --single-cluster | istioctl --kubeconfig=$KUBECONFIG1 install -y -f -

curl -O https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/expose-istiod.yaml
kubectl --kubeconfig=$KUBECONFIG1 apply -n istio-system -f expose-istiod.yaml

kubectl --kubeconfig=$KUBECONFIG1 create namespace "${VM_NAMESPACE}"
kubectl --kubeconfig=$KUBECONFIG1 create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"

```bash
cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF
```

istioctl --kubeconfig=$KUBECONFIG1 x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"


kubectl --kubeconfig=$KUBECONFIG1 apply -k vl3-dns
k1 apply -f sample-ns.yaml

```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f ubuntu.yaml
```

```bash
UBUNTU_POD_NAME=$(kubectl --kubeconfig=$KUBECONFIG2 get pods -l app=ubuntu -n default --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=1m pod -l app=ubuntu
```bash
kubectl --kubeconfig=$KUBECONFIG2 cp "$WORK_DIR" $UBUNTU_POD_NAME:/vm-dir -c ubuntu
```


```bash
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt update
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo
```

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl hello2.my-vl3-network:5000/hello 2> /dev/null
