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

```bash
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >2-istio-nsm.pcap &
sleep 1
k2 apply -k ubuntu-hosts-2
sleep 0.5
k2 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
sleep 1
kill -2 $!
sleep 1
k2 delete -k ubuntu-hosts-2
tshark -r 2-istio-nsm.pcap
```

```bash
k1 apply -f sample-ns.yaml
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG1 -n sample wait --for=condition=ready --timeout=2m pod -l app=helloworld
```

redeploy ubuntu
```bash
k2 apply -k ubuntu-hosts-2
sleep 0.5
k2 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- apk add tcpdump
```

Check mtls (should be disabled by default)
```bash
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >3-mtls-default.pcap &
sleep 1
k2 -n vl3-test exec deployments/ubuntu-deployment -c istio-proxy -- curl helloworld.my-vl3-network:5000/hello -s
sleep 1
kill -2 $!
tshark -r 3-mtls-default.pcap | grep HTTP
```

enable mtls
```bash
k1 apply -f mtls-service-entry-hw1.yaml
k1 apply -f mtls-dest-rule.yaml
```

Check mtls again
```bash
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -i lo -U -w - >3-mtls-enable-lo.pcap &
sleep 1
k2 -n vl3-test exec deployments/ubuntu-deployment -c istio-proxy -- curl helloworld.my-vl3-network:5000/hello -s
sleep 1
kill -2 $!
! tshark -r 3-mtls-enable-lo.pcap | grep HTTP
```
Fails for some reason

```bash
k2 -n vl3-test exec deployments/ubuntu-deployment -c istio-proxy -- curl 'localhost:15000/config_dump?include_eds' >envoy-config-dump-w-eds-mtls.json
! cat envoy-config-dump-w-eds-mtls.json | grep PassthroughCluster172
```


