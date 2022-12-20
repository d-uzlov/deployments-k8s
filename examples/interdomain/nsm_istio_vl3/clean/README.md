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
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1 --set meshConfig.accessLogFile=/dev/stdout
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- apk add tcpdump
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

fix connection:
```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f ubuntu-2.yaml
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=2m pod -l app=ubuntu-2
```

Get istio config
```bash
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- ip a
```
ingressIP - copy from nsm interface ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
rm -rf ubuntu-standard/istio-vm-configs
rm -rf ubuntu-hosts/istio-vm-configs
cp -r "${WORK_DIR}" ubuntu-standard/istio-vm-configs
cp -r "${WORK_DIR}" ubuntu-hosts/istio-vm-configs
```

```bash
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >1-istio-standard.pcap &
sleep 1
k1 apply -k ubuntu-standard
sleep 0.5
k1 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
sleep 1
kill -2 $!
sleep 1
k1 delete -k ubuntu-standard
tshark -r 1-istio-standard.pcap
```

```bash
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >1-istio-nsm.pcap &
sleep 1
k1 apply -k ubuntu-hosts
sleep 0.5
k1 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
sleep 1
kill -2 $!
sleep 1
k1 delete -k ubuntu-hosts
tshark -r 1-istio-nsm.pcap
```

k2 exec -n vl3-test deployments/greeting -c cmd-nsc -- ip a | grep nsm
k2 exec -n vl3-test deployments/greeting -c cmd-nsc -- ping 172.16.0.2 -c4
k2 exec -n vl3-test deployments/greeting -c cmd-nsc -- apk add netcat-openbsd
k2 exec -n vl3-test deployments/greeting -c cmd-nsc -- nc -v 172.16.0.2 15012
ps -aef --forest

```bash
istioctl kube-inject -f ubuntu.yaml >greeting/ubuntu-ic.yaml
```
