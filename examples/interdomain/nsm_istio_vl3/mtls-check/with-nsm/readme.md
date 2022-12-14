1. install interdomain 

2. install vl3
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -k ./vl3-dns
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=1m pod -l app=nse-vl3-vpp
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=1m pod -l app=vl3-ipam
```

install namespace
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
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm -rf /usr/local/go
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tar -C /usr/local -xzf go1.19.4.linux-amd64.tar.gz
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo sh -c 'GOBIN=/usr/local/bin /usr/local/go/bin/go install github.com/go-delve/delve/cmd/dlv@v1.20.0'
```

Rebuild istio proxy-agent:
Download istio repository and execute:
sudo make build-linux DEBUG=1

Copy result binary from `<istio-repo>/out/linux_amd64/pilot-agent` into `./pilot-agent-debug`

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
# kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm /usr/local/bin/istio-start.sh
# kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cp /vm-dir/istio-start.sh /usr/local/bin/istio-start.sh
# kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cp /vm-dir/pilot-agent-debug /usr/local/bin/pilot-agent
# kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- chmod +x /usr/local/bin/istio-start.sh
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
tshark -r 10-mtls.pcap | grep HTTP
```

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

Dump envoy state:
```bash
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/certs' > envoy-certs.json
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/clusters?format=json' >envoy-clusters.json
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/config_dump?include_eds' >envoy-config-dump-w-eds.json
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/listeners?format=json' >envoy-listeners.json
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/server_info' >envoy-server-info.json
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/clusters?format=json' >envoy-clusters.json
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/stats?format=json' >envoy-stats.json
k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/stats?format=json&usedonly' >envoy-stats-usedonly.json
```

k2 exec deployments/ubuntu-deployment -c ubuntu -- curl 'localhost:15000/config_dump?include_eds' >envoy-config-dump-w-eds-mtls.json
cat envoy-config-dump-w-eds-mtls.json | grep PassthroughCluster172