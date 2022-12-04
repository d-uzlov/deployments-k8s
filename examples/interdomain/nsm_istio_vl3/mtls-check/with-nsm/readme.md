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

```bash
k --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt --yes install tcpdump
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




задеплоить на реальной вм
снять дамп ваейршарка


Experiments:

Check without istio:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >1-http.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello 2> /dev/null
sleep 1
kill -2 $!
```
Result: only DNS queries.

Check after installing istio:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >2-istio.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello 2> /dev/null
sleep 1
kill -2 $!
```
Result: only DNS queries.
Same as without istio.

Check with istio, without DNS:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >3-istio-ip.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!
```
Result: nothing captured.

Check ping, before installing istio:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >5-nsm-ping.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- ping 172.18.0.5 -c4
sleep 1
kill -2 $!
```
Result: pings captured.

Check ping, after installing istio:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >6-istio-ping.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- ping 172.18.0.5 -c4
sleep 1
kill -2 $!
```
Result: pings captured.
Same as without istio.

------------------

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





enable istio telemetry:
```bash
kubectl apply -n istio-system --kubeconfig $KUBECONFIG1 -f - <<EOF
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
    - providers:
      - name: envoy
EOF
```
