k1 - kubectl --kubeconfig=$KUBECONFIG1  

```bash
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1
```

```bash
k1 apply -f ubuntu.yaml
```
```bash
VM_APP="vm-app"
VM_NAMESPACE="vm-ns"
WORK_DIR="/Users/thetadr/work/msm/deployments-k8s/examples/interdomain/nsm_istio_vl3/mtls-check/without-nsm/istio-vm-configs"
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

# 10.244.1.2 - istiod ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=10.244.1.2
```

```bash
k1 cp ./istio-vm-configs ubuntu-deployment-7575859557-4mwgf:/vm-dir -c ubuntu
```

```bash
k1 exec -it ubuntu-deployment-7575859557-4mwgf -c ubuntu -- bash
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
k1 exec ubuntu-deployment-7575859557-4mwgf -c ubuntu --  tcpdump -w - | wireshark -k -i -
```


```bash
curl helloworld.sample.svc:5000/hello
curl helloworld.sample.svc:5000/hello
```


