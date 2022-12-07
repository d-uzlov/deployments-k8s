# Basic setup 
1. install interdomain 

2. install vl3
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -k ./vl3-dns
```

2. install namespace
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -f istio-namespace.yaml
```

3. install istio
```bash
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1
```

4. Prepare configuration for istio
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
kubectl --kubeconfig=$KUBECONFIG1 create namespace "${VM_NAMESPACE}"
kubectl --kubeconfig=$KUBECONFIG1 create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
```

Get istio config
```bash
kubectl --kubeconfig $KUBECONFIG1 exec -n istio-system deployments/istiod -c cmd-nsc -- ip a
```
ingressIP - copy from nsm interface ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
```

## To run istio-proxy inside ubuntu container follow this:

Based on example from: [https://istio.io/latest/docs/setup/install/virtual-machine/](https://istio.io/latest/docs/setup/install/virtual-machine/)

1. apply ubuntu
```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f ubuntu.yaml
```

```bash
UBUNTU_POD_NAME=$(kubectl --kubeconfig=$KUBECONFIG2 get pods -l app=ubuntu -n default --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```
```bash
kubectl --kubeconfig=$KUBECONFIG2 cp "$WORK_DIR" $UBUNTU_POD_NAME:/vm-dir -c ubuntu
```

```bash
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt update
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo
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

check that ubuntu is visible to istiod
```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG1
```

```bash
k1 apply -f sample-ns.yaml
```

## To run manual istio-proxy container follow this:

2. Copy/paste istiod pod name
```bash
ISTIOD_NAME=
```
```bash
kubectl --kubeconfig=$KUBECONFIG1 exec -n istio-system $ISTIOD_NAME -c cmd-nsc -- ip a
```

3. ingressIP - istiod ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
```

4. Copy-paste info from WORK_DIR to configmap values in [server.yaml](./greeting/server.yaml)
5. Start the deployment
```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -k ./greeting/
```
6. Check logs
```bash
kubectl --kubeconfig=$KUBECONFIG2 logs deployment.apps/greeting -c istio-proxy
```
