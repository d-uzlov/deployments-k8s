
```bash
k --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt install tcpdump
```

```bash
k2 exec ubuntu-deployment-99cf8d8f7-5g2qm -c ubuntu --  tcpdump -w - | wireshark -k -i -
```

```bash
k --kubeconfig $KUBECONFIG2 exec -it deployments/ubuntu-deployment -c ubuntu -- bash
```


```bash
curl helloworld.my-vl3-network:5000/hello
curl helloworld.my-vl3-network:5000/hello
```

ubuntu container had no new packets in wireshark after curl

cmd-nsc packets in [nsm-interface-ubuntu-curl.pcapng](./nsm-interface-ubuntu-curl.pcapng)
