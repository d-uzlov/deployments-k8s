
Sample of running these commands can be found here: [readme-run-sample](./readme-run-sample.md)

---

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
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- ip a | grep 172.16.0.2/32
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

deploy service
```bash
k1 apply -f sample-ns.yaml
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG1 -n sample wait --for=condition=ready --timeout=2m pod -l app=helloworld
k1 exec -n sample deployments/helloworld-v1 -c cmd-nsc -- ip a | grep 172.16.0.3/32
```

fix connection:
```bash
# not needed currently
# kubectl --kubeconfig=$KUBECONFIG2 apply -f ubuntu-2.yaml
# sleep 0.5
# kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=2m pod -l app=ubuntu-2
# kubectl --kubeconfig=$KUBECONFIG2 delete -f ubuntu-2.yaml
```

Deploy test pod on second cluster:
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
rm -rf ubuntu-hosts-2/istio-vm-configs
cp -r "${WORK_DIR}" ubuntu-hosts-2/istio-vm-configs
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-hosts-2-dep.pcap &
sleep 0.5
k2 apply -k ubuntu-hosts-2
sleep 0.5
k2 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
sleep 3
kill -2 $!
k2 logs -n vl3-test deployments/ubuntu-deployment istio-proxy >logs-ubuntu-hosts-2-istio.log
k2 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- apt update -qq >/dev/null 2>/dev/null
k2 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- apt install curl tcpdump -y -qq >/dev/null 2>/dev/null
```

Use tcpdump with default settings:
```bash
k1 delete -f mtls-service-entry-hw1.yaml
k1 delete -f mtls-dest-rule.yaml
k2 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-hosts-2-curl-http.pcap &
sleep 0.5
k2 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -sS
sleep 1
kill -2 $!
```

Use tcpdump with manual service entry:
```basg
k1 apply -f mtls-service-entry-hw1.yaml
k1 apply -f mtls-dest-rule.yaml
sleep 0.5
k2 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-hosts-2-curl-mtls.pcap &
sleep 0.5
k2 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -sS
sleep 1
kill -2 $!
```

Check tcpdump content:
```bash
tshark -r dump-hosts-2-curl-http.pcap | grep 'GET /hello' && ! tshark -r dump-hosts-2-curl-mtls.pcap | grep HTTP
```
Result: mtls works

Cleanup:
```bash
k1 delete -f mtls-service-entry-hw1.yaml
k1 delete -f mtls-dest-rule.yaml
k2 delete -k ubuntu-hosts-2

rm dump-hosts-2-dep.pcap
rm dump-hosts-2-curl-http.pcap
rm dump-hosts-2-curl-mtls.pcap
rm logs-ubuntu-hosts-2-istio.log
```
