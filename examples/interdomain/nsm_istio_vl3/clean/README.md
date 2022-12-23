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
kubectl --kubeconfig=$KUBECONFIG2 apply -f ubuntu-2.yaml
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=2m pod -l app=ubuntu-2
kubectl --kubeconfig=$KUBECONFIG2 delete -f ubuntu-2.yaml
```

```bash
k1 apply -k ubuntu-vanilla
sleep 0.5
k1 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
k1 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- apt update -qq
k1 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- apt install curl tcpdump -y -qq

k1 delete -f mtls-service-entry-hw1.yaml
k1 delete -f mtls-dest-rule.yaml
sleep 0.25
k1 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-vanilla-curl-http.pcap &
sleep 0.5
k1 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -sS
sleep 1
kill -2 $!

k1 apply -f mtls-service-entry-hw1.yaml
k1 apply -f mtls-dest-rule.yaml
sleep 0.25
k1 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- tcpdump -f '!icmp' -i any -U -w - >dump-vanilla-curl-mtls-any.pcap &
sleep 1
k1 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -sS
sleep 2
kill -2 $!

k1 delete -f mtls-service-entry-hw1.yaml
k1 delete -f mtls-dest-rule.yaml
sleep 0.5
k1 delete -k ubuntu-vanilla
tshark -r dump-vanilla-curl-http.pcap | grep HTTP
# ! tshark -r dump-vanilla-curl-mtls.pcap | grep HTTP && tshark -r dump-vanilla-curl-mtls.pcap | grep TCP
```

```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
rm -rf ubuntu-standard/istio-vm-configs
cp -r "${WORK_DIR}" ubuntu-standard/istio-vm-configs
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-standard-dep.pcap &
sleep 0.5
k1 apply -k ubuntu-standard
sleep 0.5
k1 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
sleep 3
kill -2 $!
k2 logs -n vl3-test deployments/ubuntu-deployment istio-proxy >logs-ubuntu-standard-istio.log
k1 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- apt update -qq
k1 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- apt install curl tcpdump -y -qq

k1 delete -f mtls-service-entry-hw1.yaml
k1 delete -f mtls-dest-rule.yaml
k1 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-standard-curl-http.pcap &
sleep 0.5
k1 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -s
sleep 0.5
kill -2 $!

k1 apply -f mtls-service-entry-hw1.yaml
k1 apply -f mtls-dest-rule.yaml
sleep 0.5
k1 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-standard-curl-mtls.pcap &
sleep 0.5
k1 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -s
sleep 0.5
kill -2 $!

k1 delete -f mtls-service-entry-hw1.yaml
k1 delete -f mtls-dest-rule.yaml
sleep 0.5
k1 delete -k ubuntu-standard
tshark -r dump-standard-curl-http.pcap | grep HTTP
# ! tshark -r dump-standard-curl-mtls.pcap | grep HTTP && tshark -r dump-standard-curl-mtls.pcap | grep TCP
```

Get istio config
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
# sed -i '' 's/15012/15010/' "${WORK_DIR}/mesh.yaml"
rm -rf ubuntu-standard/istio-vm-configs
cp -r "${WORK_DIR}" ubuntu-standard/istio-vm-configs
rm -rf ubuntu-hosts/istio-vm-configs
cp -r "${WORK_DIR}" ubuntu-hosts/istio-vm-configs
```

```bash
time k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >4-istio-tcpdump-1-nsm.pcap &
sleep 1
k1 apply -k ubuntu-hosts
sleep 0.5
k1 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
k1 exec -n vl3-test deployments/ubuntu-deployment -c cmd-nsc -- apk add tcpdump
k1 exec -n vl3-test deployments/ubuntu-deployment -c cmd-nsc -- apk add curl
k1 exec -n vl3-test deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >5-istio-hosts-http.pcap &
sleep 0.5
k1 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- curl helloworld.my-vl3-network:5000/hello -s
sleep 0.5
kill -2 $!
k1 apply -f mtls-service-entry-hw1.yaml
k1 apply -f mtls-dest-rule.yaml
sleep 0.5
k1 exec -n vl3-test deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >5-istio-hosts-mtls.pcap &
sleep 0.5
k1 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- curl helloworld.my-vl3-network:5000/hello -s
sleep 0.5
kill -2 $!
k1 delete -f mtls-service-entry-hw1.yaml
k1 delete -f mtls-dest-rule.yaml
sleep 0.5
k1 delete -k ubuntu-hosts
tshark -r 5-istio-hosts-http.pcap | grep HTTP
! tshark -r 5-istio-hosts-mtls.pcap | grep HTTP
```

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
tshark -r dump-hosts-2-dep.pcap

k2 exec -n vl3-test deployments/ubuntu-deployment -c cmd-nsc -- apk add tcpdump
k2 exec -n vl3-test deployments/ubuntu-deployment -c cmd-nsc -- apk add curl
k2 exec -n vl3-test deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >dump-hosts-2-curl-http.pcap &
sleep 0.5
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- curl helloworld.my-vl3-network:5000/hello -sS
sleep 0.5
kill -2 $!
k1 apply -f mtls-service-entry-hw1.yaml
k1 apply -f mtls-dest-rule.yaml
sleep 0.5
k2 exec -n vl3-test deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >dump-hosts-2-curl-mtls.pcap &
sleep 0.5
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- curl helloworld.my-vl3-network:5000/hello -sS
sleep 0.5
kill -2 $!
k1 delete -f mtls-service-entry-hw1.yaml
k1 delete -f mtls-dest-rule.yaml
sleep 0.5
k2 delete -k ubuntu-hosts-2
tshark -r dump-hosts-2-curl-http.pcap | grep HTTP
! tshark -r dump-hosts-2-curl-mtls.pcap | grep HTTP
```

```bash
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -i nsm-1 -U -w - >2-istio-nsm-vmlike.pcap &
sleep 1
k2 apply -k ubuntu-hosts-2-vmlike
sleep 0.5
k2 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
sleep 1
kill -2 $!
sleep 1
k2 delete -k ubuntu-hosts-2-vmlike
tshark -r 2-istio-nsm-vmlike.pcap
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
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- curl helloworld.my-vl3-network:5000/hello -s
sleep 1
kill -2 $!
tshark -r 3-mtls-default.pcap | grep HTTP
```

Check mtls again
```bash
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -i lo -U -w - >3-mtls-enable-lo.pcap &
sleep 1
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- curl helloworld.my-vl3-network:5000/hello -s
sleep 1
kill -2 $!
! tshark -r 3-mtls-enable-lo.pcap | grep HTTP
```
Fails for some reason

```bash
k2 -n vl3-test exec deployments/ubuntu-deployment -c cmd-nsc -- curl 'localhost:15000/config_dump?include_eds' >envoy-config-dump-w-eds-mtls.json
! cat envoy-config-dump-w-eds-mtls.json | grep PassthroughCluster172
```


