# Test memif to IP to kernel connection

This example shows that NSC and NSE on the different nodes could find and work with each other.


NSC is using the `memif` mechanism to connect to its local forwarder.
NSE is using the `kernel` mechanism to connect to its local forwarder.
Forwarders are using the `IP` payload to connect with each other.

## Requires

Make sure that you have completed steps from [basic](../../basic) or [memory](../../memory) or [ipsec mechanism](../../ipsec_mechanism) setup.

## Run

Create test namespace:
```bash
kubectl create ns ns-memif2ip2kernel
```

Deploy NSC and NSE:
```bash
kubectl apply -k https://github.com/networkservicemesh/deployments-k8s/examples/use-cases/Memif2IP2Kernel?ref=3c972c5e0ac1d8d587af4ddb271c19f170942f9b
```

Wait for applications ready:
```bash
kubectl wait --for=condition=ready --timeout=1m pod -l app=nsc-memif -n ns-memif2ip2kernel
```
```bash
kubectl wait --for=condition=ready --timeout=1m pod -l app=nse-kernel -n ns-memif2ip2kernel
```

Find NSC and NSE pods by labels:
```bash
NSC=$(kubectl get pods -l app=nsc-memif -n ns-memif2ip2kernel --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```
```bash
NSE=$(kubectl get pods -l app=nse-kernel -n ns-memif2ip2kernel --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

Ping from NSC to NSE:
```bash
result=$(kubectl exec "${NSC}" -n "ns-memif2ip2kernel" -- vppctl ping 172.16.1.100 repeat 4)
echo ${result}
! echo ${result} | grep -E -q "(100% packet loss)|(0 sent)|(no egress interface)"
```

Ping from NSE to NSC:
```bash
kubectl exec ${NSE} -n ns-memif2ip2kernel -- ping -c 4 172.16.1.101
```

## Cleanup

Delete ns:
```bash
kubectl delete ns ns-memif2ip2kernel
```