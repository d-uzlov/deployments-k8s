k1 - kubectl --kubeconfig=$KUBECONFIG1

```bash
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1
```

```bash
k --kubeconfig=$KUBECONFIG1 apply -f ubuntu.yaml
```

```bash
k --kubeconfig=$KUBECONFIG1 apply -f sample-ns.yaml
```

```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG1
```

Install curl into ubuntu container
```bash
k --kubeconfig=$KUBECONFIG1 exec -n ubuntu-ns deployments/ubuntu-deployment -c ubuntu -- apt update
k --kubeconfig=$KUBECONFIG1 exec -n ubuntu-ns deployments/ubuntu-deployment -c ubuntu -- apt --yes install tcpdump curl
```

Disable mTLS:
```bash
kubectl apply -n istio-system --kubeconfig $KUBECONFIG1 -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: DISABLE
EOF
```

Get dump:
```bash
k --kubeconfig=$KUBECONFIG1 exec -n ubuntu-ns deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >1-http.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG1 exec -n ubuntu-ns deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello -s
sleep 1
kill -2 $!
```

In the dump file there will be DNS query and plain http data `HTTP/1.1 200 OK  (text/html)`.
For example: `Hello version: v1, instance: helloworld-v1-78b9f5c87f-lfxlr`.

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

```bash
k --kubeconfig=$KUBECONFIG1 exec -n ubuntu-ns deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >2-mtls.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG1 exec -n ubuntu-ns deployments/ubuntu-deployment -c ubuntu -- curl helloworld.sample.svc:5000/hello -s
sleep 1
kill -2 $!
```

In the dump file there will be DNS query and TCP/RSL encrypted data.
