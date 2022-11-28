k2 - kubectl --kubeconfig=$KUBECONFIG2



set basic

```bash
k1 apply -k ./vl3-dns
```

```bash
k1 apply -f istio-namespace.yaml
```

```bash
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1
```

```bash
k1 apply -f ubuntu.yaml
```
```bash
VM_APP="vm-app"
VM_NAMESPACE="vm-ns"
WORK_DIR="/Users/thetadr/work/msm/deployments-k8s/examples/interdomain/nsm_istio_vl3/mtls-check/with-nsm-one-cluster/istio-vm-configs"
SERVICE_ACCOUNT="serviceaccountvm"
CLUSTER_NETWORK=""
VM_NETWORK=""
CLUSTER="Kubernetes"
```

```bash
k1 create namespace "${VM_NAMESPACE}"
k1 create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
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

get nsm ip
```bash
k1 exec -n istio-system istiod-cd9c68bc-w42xj -c cmd-nsc -- ip a
```


ingressIP - istiod ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
```

```bash
k1 cp ./istio-vm-configs ubuntu-deployment-f98b7685b-428c6:/vm-dir -c ubuntu
```

```bash
k1 exec -it ubuntu-deployment-f98b7685b-428c6 -c ubuntu -- bash
```

```bash
cd vm-dir
```

```bash
apt update
apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo

sudo mkdir -p /etc/certs
sudo cp root-cert.pem /etc/certs/root-cert.pem


sudo  mkdir -p /var/run/secrets/tokens
sudo cp istio-token /var/run/secrets/tokens/istio-token


curl -LO https://storage.googleapis.com/istio-release/releases/1.15.2/deb/istio-sidecar.deb
sudo dpkg -i istio-sidecar.deb
```

```bash
sudo cp cluster.env /var/lib/istio/envoy/cluster.env

sudo cp mesh.yaml /etc/istio/config/mesh

sudo sh -c 'cat hosts >> /etc/hosts'
```

```bash
sudo mkdir -p /etc/istio/proxy
sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
```

```bash
cd /
sudo systemctl start istio
cat /var/log/istio/istio.log
```

in new terminal window

```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG1
```

```bash
k1 apply -f sample-ns.yaml
```

from ubuntu container

```bash
apt install tcpdump
```

```bash
k2 exec ubuntu-deployment-f98b7685b-bn625 -c ubuntu --  tcpdump -w - | wireshark -k -i -
```


```bash
curl helloworld.my-vl3-network:5000/hello
curl helloworld.my-vl3-network:5000/hello
```

ubuntu container had no new packets in wireshark after curl






Had to switch to interdomain setup due this
```
2022-11-23T05:15:36.551450Z	info	JWT policy is third-party-jwt
2022-11-23T05:15:36.551469Z	info	using credential fetcher of JWT type in cluster.local trust domain
2022-11-23T05:15:36.851755Z	info	Opening status port 15020
2022-11-23T05:15:36.851876Z	info	dns	Starting local udp DNS server on 127.0.0.1:15053
2022-11-23T05:15:36.852175Z	info	dns	Starting local tcp DNS server on 127.0.0.1:15053
2022-11-23T05:15:36.852253Z	info	Workload SDS socket not found. Starting Istio SDS Server
2022-11-23T05:15:36.852283Z	info	CA Endpoint istiod.istio-system.svc:15012, provider Citadel
2022-11-23T05:15:36.852309Z	info	Using CA istiod.istio-system.svc:15012 cert with certs: ./etc/certs/root-cert.pem
2022-11-23T05:15:36.852463Z	info	citadelclient	Citadel client using custom root cert: ./etc/certs/root-cert.pem
2022-11-23T05:15:36.876415Z	info	ads	All caches have been synced up in 342.117295ms, marking server ready
2022-11-23T05:15:36.877168Z	info	xdsproxy	Initializing with upstream address "istiod.istio-system.svc:15012" and cluster "Kubernetes"
2022-11-23T05:15:36.877246Z	info	sds	Starting SDS grpc server
2022-11-23T05:15:36.877898Z	info	starting Http service at 127.0.0.1:15004
2022-11-23T05:15:36.880443Z	info	Pilot SAN: [istiod.istio-system.svc]
2022-11-23T05:15:36.883883Z	info	Starting proxy agent
2022-11-23T05:15:36.883909Z	info	starting
2022-11-23T05:15:36.883932Z	info	Envoy command: [-c etc/istio/proxy/envoy-rev.json --drain-time-s 45 --drain-strategy immediate --parent-shutdown-time-s 60 --local-address-ip-version v4 --file-flush-interval-msec 1000 --disable-hot-restart --log-format %Y-%m-%dT%T.%fZ	%l	envoy %n	%v -l warning --component-log-level misc:error --concurrency 2]
2022-11-23T05:15:36.911860Z	warn	citadelclient	cannot parse the cert chain, using token instead: failed to read the cert, error is open etc/certs/cert-chain.pem: no such file or directory
2022-11-23T05:15:37.027165Z	info	cache	generated new workload certificate	latency=150.087453ms ttl=23h59m59.972853129s
2022-11-23T05:15:37.027260Z	info	cache	Root cert has changed, start rotating root cert
2022-11-23T05:15:37.027290Z	info	ads	XDS: Incremental Pushing:0 ConnectedEndpoints:0 Version:
2022-11-23T05:15:37.027811Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.97219611s
2022-11-23T05:15:37.371188Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2022-11-23T05:15:47.807041Z	warn	xdsproxy	upstream [2] terminated with unexpected error rpc error: code = Unavailable desc = error reading from server: read tcp 172.16.0.3:40964->172.16.0.2:15012: read: connection timed out
2022-11-23T05:15:47.967118Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
```

