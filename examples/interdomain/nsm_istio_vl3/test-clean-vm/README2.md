
kubectl --kubeconfig=$KUBECONFIG1 apply -k vl3-dns
k1 apply -f sample-ns.yaml

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl -s helloworld.my-vl3-network:5000/hello

Prepare configuration for istio
```bash
WORK_DIR="$(git rev-parse --show-toplevel)/examples/interdomain/nsm_istio_vl3/clean/istio-vm-configs"
VM_APP="vm-app"
VM_NAMESPACE="vm-ns"
SERVICE_ACCOUNT="serviceaccountvm"
CLUSTER_NETWORK=""
VM_NETWORK=""
CLUSTER="Kubernetes"
```

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
  meshConfig:
    accessLogFile: /dev/stdout
EOF
```
```bash
istioctl install -f vm-cluster.yaml --kubeconfig=$KUBECONFIG1 -y
```

```bash
curl -O https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/gen-eastwest-gateway.sh
chmod +x gen-eastwest-gateway.sh
./gen-eastwest-gateway.sh --single-cluster | istioctl --kubeconfig=$KUBECONFIG1 install -y -f -
```

```bash
curl -O https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/expose-istiod.yaml
kubectl --kubeconfig=$KUBECONFIG1 apply -n istio-system -f expose-istiod.yaml
```

```bash
kubectl --kubeconfig=$KUBECONFIG1 create namespace "${VM_NAMESPACE}"
kubectl --kubeconfig=$KUBECONFIG1 create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
```

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

```bash
istioctl --kubeconfig=$KUBECONFIG1 x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"
```








```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f /Users/daniluzlov/work/deployments-k8s/examples/interdomain/nsm_istio_vl3/test-clean-vm/ubuntu.yaml
kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=1m pod -l app=ubuntu
```

```bash
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt update
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo
```


```bash
UBUNTU_POD_NAME=$(kubectl --kubeconfig=$KUBECONFIG2 get pods -l app=ubuntu -n default --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

```bash
kubectl --kubeconfig=$KUBECONFIG2 cp "$WORK_DIR" $UBUNTU_POD_NAME:/vm-dir -c ubuntu
```

```bash
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /etc/certs
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/root-cert.pem /etc/certs/root-cert.pem
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /var/run/secrets/tokens
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/istio-token /var/run/secrets/tokens/istio-token
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo curl -LO https://storage.googleapis.com/istio-release/releases/1.15.2/deb/istio-sidecar.deb
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo dpkg -i istio-sidecar.deb
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/cluster.env /var/lib/istio/envoy/cluster.env
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/mesh.yaml /etc/istio/config/mesh
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo sh -c 'cat /vm-dir/hosts >> /etc/hosts'
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /etc/istio/proxy
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo systemctl start istio
sleep 5
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cat /var/log/istio/istio.log
```

kubectl --kubeconfig $KUBECONFIG1 create namespace sample
kubectl --kubeconfig $KUBECONFIG1 label namespace sample istio-injection=enabled
kubectl --kubeconfig $KUBECONFIG1 apply -n sample -f samples/helloworld/helloworld.yaml



kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello



k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes tcpdump curl
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i any -U -w - >1-dump.pcap &
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 10.96.22.180:5000/hello --max-time 1
k1 -n sample logs deployments/helloworld-v1 istio-proxy > hello-envoy.log
kill -2 $!


kubectl --kubeconfig $KUBECONFIG1 exec deployments/ubuntu-deployment -c ubuntu -- apt update
kubectl --kubeconfig $KUBECONFIG1 exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo
kubectl --kubeconfig $KUBECONFIG1 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello

kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello
