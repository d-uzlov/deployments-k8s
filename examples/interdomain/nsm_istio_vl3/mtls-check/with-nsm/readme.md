
```bash
k --kubeconfig $KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- apt --yes install tcpdump
```

```bash
k2 exec ubuntu-deployment-99cf8d8f7-5g2qm -c ubuntu --  tcpdump -w - | wireshark -k -i -
```

```bash
k --kubeconfig $KUBECONFIG2 exec -it deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello
k --kubeconfig $KUBECONFIG2 exec -it deployments/ubuntu-deployment -c ubuntu -- bash
```


```bash
curl helloworld.my-vl3-network:5000/hello
curl helloworld.my-vl3-network:5000/hello
```

```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >1-http-ubuntu.pcap &
UBUNTU_DUMP_PID=$!
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -U -w - >1-http-cmd-nsc.pcap &
CMD_NSC_DUMP_PID=$!
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello 2> /dev/null
sleep 1
kill -2 $UBUNTU_DUMP_PID
kill -2 $CMD_NSC_DUMP_PID
```

```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >2-istio-ubuntu.pcap &
UBUNTU_DUMP_PID=$!
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -U -w - >2-istio-cmd-nsc.pcap &
CMD_NSC_DUMP_PID=$!
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello 2> /dev/null
sleep 1
kill -2 $UBUNTU_DUMP_PID
kill -2 $CMD_NSC_DUMP_PID
```

```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >3-istio-ip-ubuntu.pcap &
UBUNTU_DUMP_PID=$!
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -U -w - >3-istio-ip-cmd-nsc.pcap &
CMD_NSC_DUMP_PID=$!
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
# k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello 2> /dev/null
sleep 1
kill -2 $UBUNTU_DUMP_PID
kill -2 $CMD_NSC_DUMP_PID
```

```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >4-ping-ubuntu.pcap &
UBUNTU_DUMP_PID=$!
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -U -w - >4-ping-cmd-nsc.pcap &
CMD_NSC_DUMP_PID=$!
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- ping 172.18.0.5 -c4
sleep 1
kill -2 $UBUNTU_DUMP_PID
kill -2 $CMD_NSC_DUMP_PID
```

```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >5-nsm-ping-ubuntu.pcap &
UBUNTU_DUMP_PID=$!
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -U -w - >5-nsm-ping-cmd-nsc.pcap &
CMD_NSC_DUMP_PID=$!
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- ping 172.18.0.5 -c4
sleep 1
kill -2 $UBUNTU_DUMP_PID
kill -2 $CMD_NSC_DUMP_PID
```

```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -U -w - >6-istio-ping-ubuntu.pcap &
UBUNTU_DUMP_PID=$!
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c cmd-nsc -- tcpdump -U -w - >6-istio-ping-cmd-nsc.pcap &
CMD_NSC_DUMP_PID=$!
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- ping 172.18.0.5 -c4
sleep 1
kill -2 $UBUNTU_DUMP_PID
kill -2 $CMD_NSC_DUMP_PID
```

```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i lo -U -w - >9-pure-interfaces-lo.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i eth0 -U -w - >9-pure-interfaces-eth0.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i nsm-1 -U -w - >9-pure-interfaces-nsm-1.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!
```

```bash
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i lo -U -w - >9-istio-interfaces-lo.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i eth0 -U -w - >9-istio-interfaces-eth0.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!

k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- tcpdump -i nsm-1 -U -w - >9-istio-interfaces-nsm-1.pcap &
sleep 1
k --kubeconfig=$KUBECONFIG2 exec deployments/ubuntu-deployment -c ubuntu -- curl 172.16.0.4:5000/hello 2> /dev/null
sleep 1
kill -2 $!
```

ubuntu container had no new packets in wireshark after curl

cmd-nsc packets in [nsm-interface-ubuntu-curl.pcapng](./nsm-interface-ubuntu-curl.pcapng)
