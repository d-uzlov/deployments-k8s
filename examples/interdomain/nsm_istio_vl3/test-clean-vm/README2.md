Prerequisites:
2 clusters.
Load balancer on the first cluster.
Our interdomain setup with 2 clusters fits these requrements.

Prepare configuration for istio
```bash
WORK_DIR="$(git rev-parse --show-toplevel)/examples/interdomain/nsm_istio_vl3/test-clean-vm/istio-vm-configs"
VM_APP="vm-app"
VM_NAMESPACE="vm-ns"
SERVICE_ACCOUNT="serviceaccountvm"
CLUSTER_NETWORK=""
VM_NETWORK=""
CLUSTER="Kubernetes"
```

```bash
cat <<EOF > ./vm-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "${CLUSTER}"
      network: "${CLUSTER_NETWORK}"
  meshConfig:
    accessLogFile: /dev/stdout
EOF
```
Deploy istio:
```bash
istioctl install -f vm-cluster.yaml --kubeconfig=$KUBECONFIG1 -y
```

Add gateways:
```bash
curl -O https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/gen-eastwest-gateway.sh
chmod +x gen-eastwest-gateway.sh
./gen-eastwest-gateway.sh --single-cluster | istioctl --kubeconfig=$KUBECONFIG1 install -y -f -
curl -O https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/expose-istiod.yaml
kubectl --kubeconfig=$KUBECONFIG1 apply -n istio-system -f expose-istiod.yaml
```

Deploy ubuntu on another cluster (will be used instead of VM):
```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f ubuntu-no-ns.yaml
kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=1m pod -l app=ubuntu
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt update
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes tcpdump curl
```

Add vm config to cluster:
```bash
kubectl --kubeconfig=$KUBECONFIG1 create namespace "${VM_NAMESPACE}"
kubectl --kubeconfig=$KUBECONFIG1 create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
```
```bash
cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF
```

Generate config:
```bash
istioctl --kubeconfig=$KUBECONFIG1 x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"
```
Copy config into vm:
```bash
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- rm -rf /vm-dir
UBUNTU_POD_NAME=$(kubectl --kubeconfig=$KUBECONFIG2 get pods -l app=ubuntu -n default --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
kubectl --kubeconfig=$KUBECONFIG2 cp "$WORK_DIR" $UBUNTU_POD_NAME:/vm-dir -c ubuntu
```
Set up config in vm:
```bash
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /etc/certs
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/root-cert.pem /etc/certs/root-cert.pem
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /var/run/secrets/tokens
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/istio-token /var/run/secrets/tokens/istio-token
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo curl -LO https://storage.googleapis.com/istio-release/releases/1.16.0/deb/istio-sidecar.deb
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo dpkg -i istio-sidecar.deb
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/cluster.env /var/lib/istio/envoy/cluster.env
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sh -c "echo \"ISTIO_AGENT_FLAGS=\\\"--log_output_level=dns:debug --proxyLogLevel=trace\\\"\" >> /var/lib/istio/envoy/cluster.env"
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo cp /vm-dir/mesh.yaml /etc/istio/config/mesh
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo sh -c 'cat /vm-dir/hosts >> /etc/hosts'
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo mkdir -p /etc/istio/proxy
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- sudo systemctl start istio
sleep 5
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- cat /var/log/istio/istio.log
```
You should see messages like these in logs:
Root cert has changed, start rotating root cert
ADS: new connection for node:ubuntu-deployment-696fdf7b4-nsdhk.vm-ns-1
SDS: PUSH request for node:ubuntu-deployment-696fdf7b4-nsdhk.vm-ns resources:1 size:4.0kB resource:default

```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG1
```

Deploy sample service:
```bash
kubectl --kubeconfig $KUBECONFIG1 create namespace sample
kubectl --kubeconfig $KUBECONFIG1 label namespace sample istio-injection=enabled
curl -O https://raw.githubusercontent.com/istio/istio/release-1.16/samples/helloworld/helloworld.yaml
kubectl --kubeconfig $KUBECONFIG1 apply -n sample -f helloworld.yaml
kubectl --kubeconfig $KUBECONFIG1 -n sample delete deployment.apps/helloworld-v2
kubectl --kubeconfig=$KUBECONFIG1 -n sample wait --for=condition=ready --timeout=1m pod -l app=helloworld
```

Check istio DNS:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- nslookup helloworld.sample.svc
```
There are no issues with DNS.

Capture network:
```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i lo -U -w - >1-lo.pcap &
sleep 1
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello --max-time 1
sleep 1
kill -2 $!
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i eth0 -U -w - >1-eth0.pcap &
sleep 1
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello --max-time 1
sleep 1
kill -2 $!
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i any -U -w - >1-any.pcap &
sleep 1
kubectl --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello --max-time 1
sleep 1
kill -2 $!
```

### Check results

Curl behavior:
download animation starts but it will go on indefinitely (until some of the connection timeouts expire)

✗ k1 -n sample get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.96.137.147   <none>        5000/TCP   22m

✗ k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN group default qlen 1000
    link/tunnel6 :: brd :: permaddr f208:460:f6a3::
4: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 86:3c:e5:59:4b:6f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.0.13/24 brd 10.244.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::843c:e5ff:fe59:4b6f/64 scope link 
       valid_lft forever preferred_lft forever

✗ tshark -r 1-eth0.pcap                                                           
    1   0.000000  10.244.0.13 → 10.244.1.9   TCP 74 45592 → 5000 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=1056154564 TSecr=0 WS=128
    2   0.083122  10.244.0.13 → 10.244.1.9   TCP 74 54292 → 5000 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=1056154647 TSecr=0 WS=128
    3   1.088024  10.244.0.13 → 10.244.1.9   TCP 74 [TCP Retransmission] [TCP Port numbers reused] 54292 → 5000 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=1056155652 TSecr=0 WS=128

✗ tshark -r 1-lo.pcap  
    1   0.000000  10.244.0.13 → 127.0.0.1    DNS 107 Standard query 0x51c6 A helloworld.sample.svc.default.svc.cluster.local
    2   0.000054  10.244.0.13 → 127.0.0.1    DNS 107 Standard query 0xccc3 AAAA helloworld.sample.svc.default.svc.cluster.local
    3   0.000564   10.96.0.10 → 10.244.0.13  DNS 226 Standard query response 0x51c6 A helloworld.sample.svc.default.svc.cluster.local CNAME helloworld.sample.svc A 10.96.137.147
    4   0.000722   10.96.0.10 → 10.244.0.13  DNS 107 Standard query response 0xccc3 AAAA helloworld.sample.svc.default.svc.cluster.local
    5   0.000920  10.244.0.13 → 127.0.0.1    TCP 74 44214 → 15001 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=3529502221 TSecr=0 WS=128
    6   0.000930 10.96.137.147 → 10.244.0.13  TCP 74 5000 → 44214 [SYN, ACK] Seq=0 Ack=1 Win=65483 Len=0 MSS=65495 SACK_PERM TSval=253776617 TSecr=3529502221 WS=128
    7   0.000939  10.244.0.13 → 127.0.0.1    TCP 66 44214 → 15001 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=3529502221 TSecr=253776617
    8   0.001032  10.244.0.13 → 127.0.0.1    HTTP 161 GET /hello HTTP/1.1 
    9   0.001037 10.96.137.147 → 10.244.0.13  TCP 66 5000 → 44214 [ACK] Seq=1 Ack=96 Win=65408 Len=0 TSval=253776617 TSecr=3529502221
   10   1.001468  10.244.0.13 → 127.0.0.1    TCP 66 44214 → 15001 [FIN, ACK] Seq=96 Ack=1 Win=64256 Len=0 TSval=3529503222 TSecr=253776617
   11   1.001797 10.96.137.147 → 10.244.0.13  TCP 66 5000 → 44214 [FIN, ACK] Seq=1 Ack=97 Win=65536 Len=0 TSval=253777618 TSecr=3529503222
   12   1.001814  10.244.0.13 → 127.0.0.1    TCP 66 44214 → 15001 [ACK] Seq=97 Ack=2 Win=64256 Len=0 TSval=3529503222 TSecr=253777618

It seems like tcp connection is successfully established but http service doesn't answer anything.

Corresponding log entry from ubuntu local envoy (not useful):
[2022-12-05T07:32:05.788Z] "GET /hello HTTP/1.1" 0 DC downstream_remote_disconnect - "-" 0 0 999 - "-" "curl/7.81.0" "29a2f777-744a-4b3e-acc2-72d434cceb78" "helloworld.sample.svc:5000" "-" outbound|5000||helloworld.sample.svc.cluster.local - 10.96.137.147:5000 10.244.0.13:45588 - default

Check helloworld service logs:
✗ k1 -n sample logs pods/helloworld-v1-78b9f5c87f-g5vjn istio-proxy | tail
2022-12-05T07:17:36.065638Z     info    ads     ADS: new connection for node:helloworld-v1-78b9f5c87f-g5vjn.sample-1
2022-12-05T07:17:36.066896Z     info    ads     ADS: new connection for node:helloworld-v1-78b9f5c87f-g5vjn.sample-2
2022-12-05T07:17:36.069108Z     info    cache   returned workload certificate from cache        ttl=23h59m58.930902815s
2022-12-05T07:17:36.070793Z     info    ads     SDS: PUSH request for node:helloworld-v1-78b9f5c87f-g5vjn.sample resources:1 size:4.0kB resource:default
2022-12-05T07:17:36.071638Z     info    cache   returned workload trust anchor from cache       ttl=23h59m58.928369559s
2022-12-05T07:17:36.072277Z     info    ads     SDS: PUSH request for node:helloworld-v1-78b9f5c87f-g5vjn.sample resources:1 size:1.1kB resource:ROOTCA
2022-12-05T07:17:36.433416Z     info    Readiness succeeded in 914.286436ms
2022-12-05T07:17:36.434377Z     info    Envoy proxy is ready
2022-12-05T07:48:04.075085Z     info    xdsproxy        connected to upstream XDS server: istiod.istio-system.svc:15012

It seems it didn't get the request.


Check local request with automatic proxy injection:

kubectl --kubeconfig $KUBECONFIG1 apply -f ubuntu-with-istio.yaml
kubectl --kubeconfig $KUBECONFIG1 -n ubuntu-ns wait --for=condition=ready --timeout=1m pod -l app=ubuntu
kubectl --kubeconfig $KUBECONFIG1 -n ubuntu-ns exec deployments/ubuntu-deployment -c ubuntu -- apt update
kubectl --kubeconfig $KUBECONFIG1 -n ubuntu-ns exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo
k --kubeconfig=$KUBECONFIG1 -n ubuntu-ns exec deployments/ubuntu-deployment -c ubuntu -- apt install --yes tcpdump curl

kubectl --kubeconfig $KUBECONFIG1 -n ubuntu-ns exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello

✗ k1 -n ubuntu-ns logs pods/ubuntu-deployment-696fdf7b4-qpx6j istio-proxy | tail
[2022-12-05T07:59:36.854Z] "GET /ubuntu/pool/universe/i/inetutils/inetutils-ping_2.2-2_amd64.deb HTTP/1.1" 200 - via_upstream - "-" 0 63724 164 155 "-" "Debian APT-HTTP/1.3 (2.4.8)" "dcb9edd7-72f6-4a0c-b6e1-67f6d8f86b85" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:53290 91.189.91.39:80 10.244.1.11:44356 - allow_any
[2022-12-05T07:59:37.018Z] "GET /ubuntu/pool/main/o/openldap/libldap-common_2.5.13%2bdfsg-0ubuntu0.22.04.1_all.deb HTTP/1.1" 200 - via_upstream - "-" 0 15864 158 156 "-" "Debian APT-HTTP/1.3 (2.4.8)" "39ad1fc0-a16a-4442-a7bb-2e597f1d2d8f" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:53290 91.189.91.39:80 10.244.1.11:44356 - allow_any
[2022-12-05T07:59:37.177Z] "GET /ubuntu/pool/main/c/cyrus-sasl2/libsasl2-modules_2.1.27%2bdfsg2-3ubuntu1_amd64.deb HTTP/1.1" 200 - via_upstream - "-" 0 57484 163 155 "-" "Debian APT-HTTP/1.3 (2.4.8)" "0cdddae8-1110-4410-871d-a871c07b7e80" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:53290 91.189.91.39:80 10.244.1.11:44356 - allow_any
[2022-12-05T07:59:37.341Z] "GET /ubuntu/pool/universe/d/docker-systemctl-replacement/systemctl_1.4.4181-1.1_all.deb HTTP/1.1" 200 - via_upstream - "-" 0 76608 165 155 "-" "Debian APT-HTTP/1.3 (2.4.8)" "9926a3a4-ac91-4c03-b8e4-86b40c7cefd3" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:53290 91.189.91.39:80 10.244.1.11:44356 - allow_any
[2022-12-05T07:59:48.909Z] "GET /ubuntu/pool/main/a/apparmor/libapparmor1_3.0.4-2ubuntu2.1_amd64.deb HTTP/1.1" 200 - via_upstream - "-" 0 38730 537 305 "-" "Debian APT-HTTP/1.3 (2.4.8)" "f95b4847-f65d-48c2-b8af-18c7036688b3" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:43264 91.189.91.39:80 10.244.1.11:43252 - allow_any
[2022-12-05T07:59:49.458Z] "GET /ubuntu/pool/main/d/dbus/libdbus-1-3_1.12.20-2ubuntu4.1_amd64.deb HTTP/1.1" 200 - via_upstream - "-" 0 188562 565 152 "-" "Debian APT-HTTP/1.3 (2.4.8)" "9006c645-a168-420b-b994-e2f787803b97" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:43264 91.189.91.39:80 10.244.1.11:43252 - allow_any
[2022-12-05T07:59:50.023Z] "GET /ubuntu/pool/main/d/dbus/dbus_1.12.20-2ubuntu4.1_amd64.deb HTTP/1.1" 200 - via_upstream - "-" 0 158098 233 152 "-" "Debian APT-HTTP/1.3 (2.4.8)" "818e8940-ca4e-4d85-a169-adbc653e59e6" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:43264 91.189.91.39:80 10.244.1.11:43252 - allow_any
[2022-12-05T07:59:50.257Z] "GET /ubuntu/pool/main/libp/libpcap/libpcap0.8_1.10.1-4build1_amd64.deb HTTP/1.1" 200 - via_upstream - "-" 0 145100 225 151 "-" "Debian APT-HTTP/1.3 (2.4.8)" "315eeb40-fccf-4ab2-9c25-6ee4c2ab713b" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:43264 91.189.91.39:80 10.244.1.11:43252 - allow_any
[2022-12-05T07:59:50.482Z] "GET /ubuntu/pool/main/t/tcpdump/tcpdump_4.99.1-3build2_amd64.deb HTTP/1.1" 200 - via_upstream - "-" 0 501232 406 152 "-" "Debian APT-HTTP/1.3 (2.4.8)" "a51c9f06-14f4-4efe-8ea7-23f4b5cf2c6b" "archive.ubuntu.com" "91.189.91.39:80" PassthroughCluster 10.244.1.11:43264 91.189.91.39:80 10.244.1.11:43252 - allow_any
[2022-12-05T08:01:33.196Z] "GET /hello HTTP/1.1" 200 - via_upstream - "-" 0 60 356 355 "-" "curl/7.81.0" "7c25eadf-f510-4494-8601-62f621c22325" "helloworld.sample.svc:5000" "10.244.1.9:5000" outbound|5000||helloworld.sample.svc.cluster.local 10.244.1.11:46166 10.96.137.147:5000 10.244.1.11:43492 - default

✗ k1 -n sample logs pods/helloworld-v1-78b9f5c87f-g5vjn istio-proxy | tail      
2022-12-05T07:17:36.065638Z     info    ads     ADS: new connection for node:helloworld-v1-78b9f5c87f-g5vjn.sample-1
2022-12-05T07:17:36.066896Z     info    ads     ADS: new connection for node:helloworld-v1-78b9f5c87f-g5vjn.sample-2
2022-12-05T07:17:36.069108Z     info    cache   returned workload certificate from cache        ttl=23h59m58.930902815s
2022-12-05T07:17:36.070793Z     info    ads     SDS: PUSH request for node:helloworld-v1-78b9f5c87f-g5vjn.sample resources:1 size:4.0kB resource:default
2022-12-05T07:17:36.071638Z     info    cache   returned workload trust anchor from cache       ttl=23h59m58.928369559s
2022-12-05T07:17:36.072277Z     info    ads     SDS: PUSH request for node:helloworld-v1-78b9f5c87f-g5vjn.sample resources:1 size:1.1kB resource:ROOTCA
2022-12-05T07:17:36.433416Z     info    Readiness succeeded in 914.286436ms
2022-12-05T07:17:36.434377Z     info    Envoy proxy is ready
2022-12-05T07:48:04.075085Z     info    xdsproxy        connected to upstream XDS server: istiod.istio-system.svc:15012
[2022-12-05T08:01:33.221Z] "GET /hello HTTP/1.1" 200 - via_upstream - "-" 0 60 291 282 "-" "curl/7.81.0" "7c25eadf-f510-4494-8601-62f621c22325" "helloworld.sample.svc:5000" "10.244.1.9:5000" inbound|5000|| 127.0.0.6:37527 10.244.1.9:5000 10.244.1.11:46166 outbound_.5000_._.helloworld.sample.svc.cluster.local default


Check gateway logs:

✗ k1 -n istio-system logs pods/istio-eastwestgateway-64b86b4c46-cz4dc | tail
2022-12-05T07:12:47.300768Z     info    cache   returned workload certificate from cache        ttl=23h59m59.699237133s
2022-12-05T07:12:47.300960Z     info    ads     SDS: PUSH request for node:istio-eastwestgateway-64b86b4c46-cz4dc.istio-system resources:1 size:4.0kB resource:default
2022-12-05T07:12:47.302730Z     info    ads     ADS: new connection for node:istio-eastwestgateway-64b86b4c46-cz4dc.istio-system-2
2022-12-05T07:12:47.303138Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.696867645s
2022-12-05T07:12:47.303320Z     info    ads     SDS: PUSH request for node:istio-eastwestgateway-64b86b4c46-cz4dc.istio-system resources:1 size:1.1kB resource:ROOTCA
2022-12-05T07:12:48.241299Z     info    Readiness succeeded in 1.336992917s
2022-12-05T07:12:48.241700Z     info    Envoy proxy is ready
[2022-12-05T07:14:23.958Z] "- - -" 0 - - - "-" 56978 465841 1684512 - "-" "-" "-" "-" "10.244.2.6:15012" outbound|15012||istiod.istio-system.svc.cluster.local 10.244.1.8:55142 10.244.1.8:15012 172.18.0.3:22830 istiod.istio-system.svc -
2022-12-05T07:45:05.368774Z     info    xdsproxy        connected to upstream XDS server: istiod.istio-system.svc:15012
[2022-12-05T07:14:23.802Z] "- - -" 0 - - - "-" 4916 6622 1902641 - "-" "-" "-" "-" "10.244.2.6:15012" outbound|15012||istiod.istio-system.svc.cluster.local 10.244.1.8:55140 10.244.1.8:15012 172.18.0.3:41244 istiod.istio-system.svc -

✗ k1 -n istio-system logs pods/istio-ingressgateway-6795c6c98d-f92w9 | tail 
2022-12-05T07:11:42.880514Z     info    ads     XDS: Incremental Pushing:0 ConnectedEndpoints:2 Version:
2022-12-05T07:11:42.880611Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.119391465s
2022-12-05T07:11:42.880802Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.119200704s
2022-12-05T07:11:42.881103Z     info    cache   returned workload certificate from cache        ttl=23h59m59.118902299s
2022-12-05T07:11:42.881406Z     info    ads     SDS: PUSH request for node:istio-ingressgateway-6795c6c98d-f92w9.istio-system resources:1 size:1.1kB resource:ROOTCA
2022-12-05T07:11:42.881544Z     info    ads     SDS: PUSH request for node:istio-ingressgateway-6795c6c98d-f92w9.istio-system resources:1 size:4.0kB resource:default
2022-12-05T07:11:42.881660Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.118343534s
2022-12-05T07:11:43.051987Z     info    Readiness succeeded in 501.750587ms
2022-12-05T07:11:43.052497Z     info    Envoy proxy is ready
2022-12-05T07:43:45.215777Z     info    xdsproxy        connected to upstream XDS server: istiod.istio-system.svc:15012

Seems like ingress gateway is not used at all.