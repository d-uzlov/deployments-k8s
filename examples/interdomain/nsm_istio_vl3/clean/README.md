# Basic setup 
1. install interdomain 

2. install vl3
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -k ./vl3-dns
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=1m pod -l app=nse-vl3-vpp
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=1m pod -l app=vl3-ipam
```

3. install istio
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -f istio-namespace.yaml
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1
```

4. Prepare configuration for istio
```bash
WORK_DIR="$(git rev-parse --show-toplevel)/examples/interdomain/nsm_istio_vl3/clean/greeting/istio-vm-configs"
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

## To run manual istio-proxy container follow this:

4. Copy-paste info from WORK_DIR to configmap values in [server.yaml](./greeting/server.yaml)
5. Start the deployment
```bash
k2 create ns vl3-test
kubectl --kubeconfig=$KUBECONFIG2 apply -k ./greeting/
```
6. Check logs
```bash
kubectl --kubeconfig=$KUBECONFIG2 -n vl3-test logs deployment.apps/greeting -c istio-proxy
```
```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG1 | grep vm-ns
```

k2 exec -n vl3-test deployments/greeting -c cmd-nsc -- ip a | grep nsm
k2 exec -n vl3-test deployments/greeting -c cmd-nsc -- ping 172.16.0.2 -c4
k2 exec -n vl3-test deployments/greeting -c cmd-nsc -- apk add netcat-openbsd
k2 exec -n vl3-test deployments/greeting -c cmd-nsc -- nc -v 172.16.0.2 15012
ps -aef --forest

```bash
istioctl kube-inject -f ubuntu.yaml >greeting/ubuntu-ic.yaml
k1 apply -k greeting
```
