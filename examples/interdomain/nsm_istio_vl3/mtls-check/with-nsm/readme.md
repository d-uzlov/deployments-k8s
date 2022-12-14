1. install interdomain

2. install vl3
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -k ./vl3-dns
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=1m pod -l app=nse-vl3-vpp
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=1m pod -l app=vl3-ipam
```

install istio
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -f istio-namespace.yaml
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1
```

Prepare configuration for istio
```bash
WORK_DIR="$(git rev-parse --show-toplevel)/examples/interdomain/nsm_istio_vl3/mtls-check/with-nsm/istio-vm-configs"
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

1. apply ubuntu
```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f ubuntu.yaml
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=2m pod -l app=ubuntu
```

```bash
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt update
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo tcpdump netcat wget git
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo curl -LO https://storage.googleapis.com/istio-release/releases/1.16.0/deb/istio-sidecar.deb
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo dpkg -i istio-sidecar.deb
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- wget -L "https://go.dev/dl/go1.19.4.linux-amd64.tar.gz"
```

Get istio config
```bash
kubectl --kubeconfig $KUBECONFIG1 exec -n istio-system deployments/istiod -c cmd-nsc -- ip a
```
ingressIP - copy from nsm interface ip.

The wollowing command will launch istio using a custom 
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
cp ./istio-start.sh istio-vm-configs/istio-start.sh
cp ./pilot-agent-debug istio-vm-configs/pilot-agent-debug
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm -rf /vm-dir
UBUNTU_POD_NAME=$(kubectl --kubeconfig=$KUBECONFIG2 get pods -l app=ubuntu -n default --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
kubectl --kubeconfig=$KUBECONFIG2 cp "$WORK_DIR" $UBUNTU_POD_NAME:/vm-dir -c ubuntu
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /etc/certs
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/root-cert.pem /etc/certs/root-cert.pem
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /var/run/secrets/tokens
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/istio-token /var/run/secrets/tokens/istio-token
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/cluster.env /var/lib/istio/envoy/cluster.env
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sh -c "echo \"ISTIO_AGENT_FLAGS=\\\"--log_output_level=all:debug --proxyLogLevel=trace\\\"\" >> /var/lib/istio/envoy/cluster.env"
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/mesh.yaml /etc/istio/config/mesh
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo sh -c 'cat /vm-dir/hosts >> /etc/hosts'
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo sh -c 'echo "172.16.0.4 helloworld.sample.svc" >> /etc/hosts'
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /etc/istio/proxy
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm -f /var/log/istio/istio.log
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm -f /var/log/istio/istio.err.log
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo systemctl start istio
sleep 5
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cat /var/log/istio/istio.log
```

You can also check the second log:
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cat /var/log/istio/istio.err.log

check that ubuntu is visible to istiod
```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG1 | grep vm-ns
```

```bash
k1 apply -f sample-ns.yaml
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG1 -n sample wait --for=condition=ready --timeout=2m pod -l app=helloworld
```

Check mtls
```bash
k2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i nsm-1 -U -w - >10-default.pcap &
sleep 1
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -s
sleep 1
kill -2 $!
tshark -r 10-default.pcap | grep HTTP
```
Expected output:
  112   1.156252   172.16.0.3 → 172.16.0.4   HTTP 990 GET /hello HTTP/1.1 
  115   1.361133   172.16.0.4 → 172.16.0.3   HTTP 1281 HTTP/1.1 200 OK  (text/html)


check envoy config
```bash
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/config_dump?include_eds' >envoy-config-dump-w-eds-default.json
cat envoy-config-dump-w-eds-default.json | grep PassthroughCluster172
```
expected output:
           "hostname": "PassthroughCluster172.16.0.4:5000"

Enforce mTLS:
```bash
kubectl apply -n istio-system --kubeconfig $KUBECONFIG1 -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
k1 apply -f mtls-service-entry-hw1.yaml
k1 apply -f mtls-dest-rule.yaml
```

```bash
k2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i nsm-1 -U -w - >10-mtls.pcap &
sleep 1
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -s
sleep 1
kill -2 $!
! tshark -r 10-mtls.pcap | grep HTTP
```
expected output: empty

check envoy config
```bash
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/config_dump?include_eds' >envoy-config-dump-w-eds-mtls.json
! cat envoy-config-dump-w-eds-mtls.json | grep PassthroughCluster172
```
expected output: empty

Disable mTLS:
```bash
k1 apply -f mtls-dest-rule-disable.yaml
kubectl apply -n istio-system --kubeconfig $KUBECONFIG1 -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: DISABLE
EOF
```

```bash
k2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i nsm-1 -U -w - >10-disable.pcap &
sleep 1
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -s
sleep 1
kill -2 $!
tshark -r 10-disable.pcap | grep HTTP
```
expected output:
   98   1.079007   172.16.0.3 → 172.16.0.4   HTTP 1053 GET /hello HTTP/1.1 
  103   1.232405   172.16.0.4 → 172.16.0.3   HTTP 1208 HTTP/1.1 200 OK  (text/html)

check envoy config
```bash
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/config_dump?include_eds' >envoy-config-dump-w-eds-disable.json
cat envoy-config-dump-w-eds-disable.json | grep PassthroughCluster172
```
expected output:
           "hostname": "PassthroughCluster172.16.0.1:53"

check istio services:
```bash
k1 -n istio-system exec deployments/istiod -c discovery -- curl 'localhost:8080/debug/registryz' >registryz.json
```
