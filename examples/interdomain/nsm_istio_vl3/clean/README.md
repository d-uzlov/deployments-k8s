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
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- ip a
```
ingressIP - copy from nsm interface ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
```

```bash
k1 apply -k greeting
k1 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
k1 -n vl3-test wait --for=condition=ready --timeout=10m pod -l app=ubuntu
k1 -n vl3-test logs deployment.apps/ubuntu-deployment -c istio-proxy >istio-2-proxy-manual-w-hosts-w-err-2.log
```

k1 cluster-info dump --output yaml --all-namespaces --output-directory dump-1-sidecar


```bash
k1 apply -k greeting
k1 -n vl3-test-w-env wait --for=condition=ready --timeout=1m pod -l app=ubuntu
k1 -n vl3-test-w-env logs deployment.apps/ubuntu-deployment -c istio-proxy >istio-proxy-manual-w-env.log
```

```bash
k1 apply -k greeting
k1 -n vl3-test-w-hosts wait --for=condition=ready --timeout=1m pod -l app=ubuntu
k1 -n vl3-test-w-hosts logs deployment.apps/ubuntu-deployment -c istio-proxy >istio-proxy-manual-w-hosts.log
```
error reading from server: read tcp 172.16.0.6:46704->172.16.0.2:15012: read: connection timed out
The following command indicates that connection should be possible:
k1 exec -n vl3-test-w-hosts deployments/ubuntu-deployment -c istio-proxy -- curl https://172.16.0.2:15012 --insecure
curl: (92) HTTP/2 stream 0 was not closed cleanly: PROTOCOL_ERROR (err 1)

```bash
k1 apply -k greeting
k1 -n vl3-test-w-hosts wait --for=condition=ready --timeout=1m pod -l app=ubuntu
k1 -n vl3-test-w-hosts logs deployment.apps/ubuntu-deployment -c istio-proxy >istio-proxy-manual-w-hosts-w-discovery.log
```

```bash
k1 apply -k greeting
k1 -n vl3-test-w-hosts-wo-init wait --for=condition=ready --timeout=1m pod -l app=ubuntu
k1 -n vl3-test-w-hosts-wo-init logs deployment.apps/ubuntu-deployment -c istio-proxy >istio-proxy-manual-w-hosts-wo-init.log
```

k1 exec -n istio-system deployments/istiod -c cmd-nsc -- apk add tcpdump

```bash
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >1-istio-nsm.pcap &
sleep 1
k1 apply -k ./greeting/
k1 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
sleep 1
kill -2 $!
sleep 1
tshark -r 1-istio-nsm.pcap
```
6. Check logs
```bash
k1 -n vl3-test logs deployment.apps/greeting -c istio-proxy
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
```
