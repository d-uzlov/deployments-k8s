k1 - kubectl --kubeconfig=$KUBECONFIG1  

```bash
istioctl  install --set profile=minimal -y --kubeconfig=$KUBECONFIG2
```

```bash
k2 apply -f ubuntu.yaml
```

```bash
k2 apply -f sample-ns.yaml
```

```bash
istioctl proxy-status --kubeconfig=$KUBECONFIG2
```

from ubuntu container

```bash
apt update
apt install tcpdump curl
```

```bash
k2 exec -n ubuntu-ns ubuntu-deployment-7575859557-d69vj -c ubuntu --  tcpdump -w - | wireshark -k -i -
```


```bash
curl helloworld.sample.svc:5000/hello
curl helloworld.sample.svc:5000/hello
```


