
```bash

kind create cluster --name istio-demo

kind get kubeconfig --name istio-demo > istio-kubeconfig.yaml
export KUBECONFIG="$PWD/istio-kubeconfig.yaml"

CLUSTER_CIDR=172.18.201.0/24

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
cat > metallb-config.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - $CLUSTER1_CIDR
EOF
kubectl apply -f metallb-config.yaml
kubectl wait --for=condition=ready pod -l app=metallb -n metallb-system

WORK_DIR="$PWD/istio-vm-configs"
VM_APP="vm-app"
VM_NAMESPACE="vm-ns"
SERVICE_ACCOUNT="serviceaccountvm"
CLUSTER_NETWORK=""
VM_NETWORK=""
CLUSTER="Kubernetes"

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
EOF

istioctl install -f vm-cluster.yaml -y

istioctl version
# client version: 1.16.0
# control plane version: 1.16.0
# data plane version: 1.16.0 (1 proxies)

curl -O -s https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/gen-eastwest-gateway.sh
chmod +x gen-eastwest-gateway.sh
./gen-eastwest-gateway.sh --single-cluster | istioctl install -y -f -
curl -O -s https://raw.githubusercontent.com/istio/istio/release-1.16/samples/multicluster/expose-istiod.yaml
kubectl apply -n istio-system -f expose-istiod.yaml

kubectl create namespace "${VM_NAMESPACE}"
kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"

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

istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"

docker run --name ubuntu --detach --entrypoint sleep ubuntu:22.04 9999999

docker cp "$WORK_DIR" ubuntu:/vm-dir -c ubuntu

docker exec ubuntu apt update
docker exec ubuntu apt install --yes curl iproute2 iptables nano dnsutils inetutils-ping systemctl sudo tcpdump
docker exec ubuntu sudo mkdir -p /etc/certs
docker exec ubuntu sudo cp /vm-dir/root-cert.pem /etc/certs/root-cert.pem
docker exec ubuntu sudo mkdir -p /var/run/secrets/tokens
docker exec ubuntu sudo cp /vm-dir/istio-token /var/run/secrets/tokens/istio-token
docker exec ubuntu sudo curl -LO https://storage.googleapis.com/istio-release/releases/1.16.0/deb/istio-sidecar.deb
docker exec ubuntu sudo dpkg -i istio-sidecar.deb
docker exec ubuntu sudo cp /vm-dir/cluster.env /var/lib/istio/envoy/cluster.env
docker exec ubuntu sh -c "echo \"ISTIO_AGENT_FLAGS=\\\"--log_output_level=dns:debug --proxyLogLevel=trace\\\"\" >> /var/lib/istio/envoy/cluster.env"
docker exec ubuntu sudo cp /vm-dir/mesh.yaml /etc/istio/config/mesh
docker exec ubuntu sudo sh -c 'cat /vm-dir/hosts >> /etc/hosts'
docker exec ubuntu sudo mkdir -p /etc/istio/proxy
docker exec ubuntu sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
docker exec ubuntu sudo systemctl start istio
sleep 5
docker exec ubuntu cat /var/log/istio/istio.log

```
