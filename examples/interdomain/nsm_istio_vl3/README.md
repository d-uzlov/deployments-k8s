1. install vl3
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

kubectl --kubeconfig=$KUBECONFIG2 apply -f greeting/server.yaml 

kubectl --kubeconfig=$KUBECONFIG1 apply -f greeting/client.yaml
