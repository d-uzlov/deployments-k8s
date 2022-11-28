# Basic setup 
1. install interdomain 

2. install vl3
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -k ./vl3-dns 
```

2. install namespace
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -f istio-namespace.yaml 
```

3. install istio
```bash
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1
```

draft state with manual isito-proxy container [server.yaml](./greeting/server.yaml) 

## To run istio-proxy inside ubuntu container follow this:
Based on example from: [https://istio.io/latest/docs/setup/install/virtual-machine/](https://istio.io/latest/docs/setup/install/virtual-machine/)

1. apply ubuntu
```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f ubuntu.yaml 
```

2. Prepare configuration for istio
set WORK_DIR to local directory (better to set absolute path) 
```bash
WORK_DIR=""
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

Copy/paste istiod pod name
```bash
ISTIOD_NAME=
```
```bash
kubectl --kubeconfig=$KUBECONFIG1 exec -n istio-system $ISTIOD_NAME -c cmd-nsc -- ip a
```


ingressIP - istiod ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
```
Copy/paste ubuntu pod name
```bash
UBUNTU_POD_NAME=
```
```bash
kubectl --kubeconfig=$KUBECONFIG2 cp ./istio-vm-configs $UBUNTU_POD_NAME:/vm-dir -c ubuntu
```

```bash
kubectl --kubeconfig=$KUBECONFIG2 exec -it $UBUNTU_POD_NAME -c ubuntu -- bash
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
```
```bash
sudo systemctl start istio
```
```bash
cat /var/log/istio/istio.log
```

in new terminal window check that ubuntu is visible to istiod

```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG1
```


## To run manual istio-proxy container follow this:


1. Prepare configuration for istio
   set WORK_DIR to local directory (better to set absolute path)
```bash
WORK_DIR=""
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

2. Copy/paste istiod pod name
```bash
ISTIOD_NAME=
```
```bash
kubectl --kubeconfig=$KUBECONFIG1 exec -n istio-system $ISTIOD_NAME -c cmd-nsc -- ip a
```

3. ingressIP - istiod ip
```bash
istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
```

4. Copy-paste info from WORK_DIR to configmap values in [server.yaml](./greeting/server.yaml)
5. Start the deployment
```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f ./greeting/server.yaml
```
6. Check logs
```bash
kubectl --kubeconfig=$KUBECONFIG2 logs deployment.apps greeting -c istio-proxy
```
