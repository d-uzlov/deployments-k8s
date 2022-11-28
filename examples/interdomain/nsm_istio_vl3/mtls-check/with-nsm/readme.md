k2 - kubectl --kubeconfig=$KUBECONFIG2

CLUSTER1_CIDR=172.18.128.0/24
CLUSTER2_CIDR=172.18.129.0/24


set interdomain

```bash
k1 apply -k ./vl3-dns
```

```bash
k1 apply -f istio-namespace.yaml
```

```bash
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1
```

Patch istio deployment if using old webhook without namespace annotations support:
kubectl --kubeconfig=$KUBECONFIG1 -n istio-system patch deployment istiod -p '{"spec": {"template":{"metadata":{"annotations":{"networkservicemesh.io":"kernel://my-vl3-network/nsm-1?dnsName=istio-cp"}}}} }'

```bash
k2 apply -f ubuntu.yaml
```
```bash
VM_APP="vm-app"
VM_NAMESPACE="vm-ns"
WORK_DIR="$(git rev-parse --show-toplevel)/examples/interdomain/nsm_istio_vl3/mtls-check/with-nsm"
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
k1 exec -n istio-system istiod-cd9c68bc-2lgsf -c cmd-nsc -- ip a
```


ingressIP - istiod ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
```

```bash
k2 cp ./istio-vm-configs ubuntu-deployment-99cf8d8f7-5g2qm:/vm-dir -c ubuntu
```

```bash
k2 exec -it ubuntu-deployment-99cf8d8f7-5g2qm -c ubuntu -- bash
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
k2 exec ubuntu-deployment-99cf8d8f7-5g2qm -c ubuntu --  tcpdump -w - | wireshark -k -i -
```


```bash
curl helloworld.my-vl3-network:5000/hello
curl helloworld.my-vl3-network:5000/hello
```

ubuntu container had no new packets in wireshark after curl

cmd-nsc packets in [nsm-interface-ubuntu-curl.pcapng](./nsm-interface-ubuntu-curl.pcapng)
