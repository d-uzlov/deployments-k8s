1. install interdomain 

2. install vl3
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -k ./vl3-dns
```

install namespace
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -f istio-namespace.yaml
```

install istio
```bash
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
kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=1m pod -l app=ubuntu
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
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /etc/istio/proxy
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm /usr/local/bin/istio-start.sh
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cp /vm-dir/istio-start.sh /usr/local/bin/istio-start.sh
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cp /vm-dir/pilot-agent-debug /usr/local/bin/pilot-agent
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- chmod +x /usr/local/bin/istio-start.sh
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm /var/log/istio/istio.log
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm /var/log/istio/istio.err.log
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo systemctl start istio
sleep 5
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cat /var/log/istio/istio.log
```

check that ubuntu is visible to istiod
```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG1 | grep vm-ns
```

```bash
k1 apply -f sample-ns.yaml
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG1 -n sample wait --for=condition=ready --timeout=1m pod -l app=helloworld
```

Start port-forwarding for debug connection:
```bash
k2 port-forward deployments/ubuntu-deployment 40000:40000
```
Open VSCode, run Go debugger with "Connect to server" settings.

Test connectivity:
```bash
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- nslookup helloworld.sample.svc
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- nslookup helloworld.my-vl3-network
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl -s helloworld.my-vl3-network:5000/hello
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello
```

------------------------

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:80/headers 2> /dev/null

➜  with-nsm git:(istio-branch) ✗ k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:80/headers 2> /dev/null
{
  "headers": {
    "Accept": "*/*", 
    "Host": "helloworld.my-vl3-network", 
    "User-Agent": "curl/7.81.0", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "750326757e9f1724", 
    "X-B3-Traceid": "c3c389d616bdf268750326757e9f1724"
  }
}

echo "ISTIO_AGENT_FLAGS=\"--log_output_level=dns:debug --proxyLogLevel=trace\"" >> /var/lib/istio/envoy/cluster.env
systemctl restart istio


Experiments:

Capture from all relevant interfaces, without istio:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i lo -U -w - >9-pure-interfaces-lo.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i eth0 -U -w - >9-pure-interfaces-eth0.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i nsm-1 -U -w - >9-pure-interfaces-nsm-1.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!
```
Result: clear text through nsm interface, nothing on other interfaces.

Capture from all relevant interfaces, with istio:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i lo -U -w - >9-istio-interfaces-lo.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i eth0 -U -w - >9-istio-interfaces-eth0.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i nsm-1 -U -w - >9-istio-interfaces-nsm-1.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!
```
Result: clear text through loopback, clear text through nsm, some DNS queries through ethernet.


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
```

Check connectivity:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello
```
Result: there is an error.


Disable mTLS enforcement (the default settings):
```bash
kubectl apply -n istio-system --kubeconfig $KUBECONFIG1 -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: PERMISSIVE
EOF
```

Check connectivity:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello
```
Result: test is working again.
