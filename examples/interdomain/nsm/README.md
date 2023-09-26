# NSM interdomain setup


This example simply show how can be deployed and configured two NSM on different clusters

## Run

Create basic NSM deployment on cluster 1:

```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -k https://github.com/d-uzlov/deployments-k8s/examples/interdomain/nsm/cluster1?ref=396a90b0927a8a77c36fdb8f2ec9f9203da74f03
```

Create basic NSM deployment on cluster 2:

```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -k https://github.com/d-uzlov/deployments-k8s/examples/interdomain/nsm/cluster2?ref=396a90b0927a8a77c36fdb8f2ec9f9203da74f03
```

Wait for NSM admission webhook on cluster 1:

```bash
kubectl --kubeconfig=$KUBECONFIG1 wait --for=condition=ready --timeout=1m pod -n nsm-system -l app=admission-webhook-k8s
```

Wait for NSM admission webhook on cluster 2:

```bash
kubectl --kubeconfig=$KUBECONFIG2 wait --for=condition=ready --timeout=1m pod -n nsm-system -l app=admission-webhook-k8s
```

## Cleanup

Cleanup NSM
```bash
WH=$(kubectl --kubeconfig=$KUBECONFIG1 get pods -l app=admission-webhook-k8s -n nsm-system --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
kubectl --kubeconfig=$KUBECONFIG1 delete mutatingwebhookconfiguration ${WH}

WH=$(kubectl --kubeconfig=$KUBECONFIG2 get pods -l app=admission-webhook-k8s -n nsm-system --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
kubectl --kubeconfig=$KUBECONFIG2 delete mutatingwebhookconfiguration ${WH}

kubectl --kubeconfig=$KUBECONFIG1 delete -k https://github.com/d-uzlov/deployments-k8s/examples/interdomain/nsm/cluster1?ref=396a90b0927a8a77c36fdb8f2ec9f9203da74f03
kubectl --kubeconfig=$KUBECONFIG2 delete -k https://github.com/d-uzlov/deployments-k8s/examples/interdomain/nsm/cluster2?ref=396a90b0927a8a77c36fdb8f2ec9f9203da74f03
```
