
This file shows my console input and output while running [readme](./README.md).

The check of the tcpdump in the end shows that we indeed enabled mtls, and so all the other functions of istio should also work.

You can find the output of the `cluster-info dump` (last command) in the same commit: [run-sample](./run-sample/).

```bash
$ kubectl --kubeconfig=$KUBECONFIG1 apply -k ./vl3-dns                                                                          [20:23:05]
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=1m pod -l app=nse-vl3-vpp
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=1m pod -l app=vl3-ipam

namespace/ns-dns-vl3 created
service/vl3-ipam created
deployment.apps/nse-vl3-vpp created
deployment.apps/vl3-ipam created
pod/nse-vl3-vpp-5d97745888-vkb9m condition met
pod/vl3-ipam-7d564f85b-9rd4q condition met



$ kubectl --kubeconfig=$KUBECONFIG1 apply -f istio-namespace.yaml                                                               [20:24:51]
istioctl install --set profile=minimal -y --kubeconfig=$KUBECONFIG1 --set meshConfig.accessLogFile=/dev/stdout
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- apk add tcpdump
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- ip a | grep 172.16.0.2/32

namespace/istio-system created
✔ Istio core installed                                                                                                                     
✔ Istiod installed                                                                                                                         
✔ Installation complete                                                                                                                    Making this installation the default for injection and validation.

Thank you for installing Istio 1.16.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/99uiMML96AmsXY5d6
fetch https://dl-cdn.alpinelinux.org/alpine/v3.17/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.17/community/x86_64/APKINDEX.tar.gz
(1/2) Installing libpcap (1.10.1-r1)
(2/2) Installing tcpdump (4.99.1-r4)
Executing busybox-1.35.0-r29.trigger
OK: 8 MiB in 17 packages
    inet 172.16.0.2/32 scope global nsm-1



$ WORK_DIR="$(git rev-parse --show-toplevel)/examples/interdomain/nsm_istio_vl3/clean/istio-vm-configs"                         [20:26:18]
VM_APP="vm-app"
VM_NAMESPACE="vm-ns"
SERVICE_ACCOUNT="serviceaccountvm"
CLUSTER_NETWORK=""
VM_NETWORK=""
CLUSTER="Kubernetes"




$ kubectl --kubeconfig=$KUBECONFIG1 create namespace "${VM_NAMESPACE}"                                                          [20:29:02]
kubectl --kubeconfig=$KUBECONFIG1 create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"

namespace/vm-ns created
serviceaccount/serviceaccountvm created
 



$ k1 apply -f sample-ns.yaml                                                                                                    [20:30:02]
sleep 0.5
kubectl --kubeconfig=$KUBECONFIG1 -n sample wait --for=condition=ready --timeout=2m pod -l app=helloworld
k1 exec -n sample deployments/helloworld-v1 -c cmd-nsc -- ip a | grep 172.16.0.3/32

namespace/sample created
service/helloworld created
deployment.apps/helloworld-v1 created
pod/helloworld-v1-5c59cf7d49-cgqv7 condition met
    inet 172.16.0.3/32 scope global nsm-1




$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --kubeconfig=$KUBECONFIG1 --ingressIP=172.16.0.2
rm -rf ubuntu-hosts-2/istio-vm-configs
cp -r "${WORK_DIR}" ubuntu-hosts-2/istio-vm-configs
k1 exec -n istio-system deployments/istiod -c cmd-nsc -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-hosts-2-dep.pcap &
sleep 0.5
k2 apply -k ubuntu-hosts-2
sleep 0.5
k2 -n vl3-test wait --for=condition=ready --timeout=1m pod -l app=ubuntu
sleep 3
kill -2 $!
k2 logs -n vl3-test deployments/ubuntu-deployment istio-proxy >logs-ubuntu-hosts-2-istio.log
k2 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- apt update -qq >/dev/null 2>/dev/null
k2 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- apt install curl tcpdump -y -qq >/dev/null 2>/dev/null

Warning: a security token for namespace "vm-ns" and service account "serviceaccountvm" has been generated and stored at "/Users/daniluzlov/work/deployments-k8s/examples/interdomain/nsm_istio_vl3/clean/istio-vm-configs/istio-token"
Configuration generation into directory /Users/daniluzlov/work/deployments-k8s/examples/interdomain/nsm_istio_vl3/clean/istio-vm-configs was successful
[1] 44171
tcpdump: listening on nsm-1, link-type RAW (Raw IP), snapshot length 262144 bytes
namespace/vl3-test created
configmap/cert-file created
deployment.apps/ubuntu-deployment created
pod/ubuntu-deployment-56bbd47db4-8vchx condition met
[1]  + 44171 interrupt  kubectl --kubeconfig /Users/daniluzlov/kind-configs/config-1 exec -n   -c  --





$ k1 delete -f mtls-service-entry-hw1.yaml                                                                                      [20:34:41]
k1 delete -f mtls-dest-rule.yaml
k2 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-hosts-2-curl-http.pcap &
sleep 0.5
k2 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -sS
sleep 1
kill -2 $!

Error from server (NotFound): error when deleting "mtls-service-entry-hw1.yaml": serviceentries.networking.istio.io "se-hw1" not found
Error from server (NotFound): error when deleting "mtls-dest-rule.yaml": destinationrules.networking.istio.io "my-vl3-network" not found
[1] 44730
tcpdump: listening on nsm-1, link-type RAW (Raw IP), snapshot length 262144 bytes
Hello version: v1, instance: helloworld-v1-5c59cf7d49-cgqv7
[1]  + 44730 interrupt  kubectl --kubeconfig /Users/daniluzlov/kind-configs/config-2 exec -n vl3-test                                      
 





$ k1 apply -f mtls-service-entry-hw1.yaml                                                                                       [20:36:40]
k1 apply -f mtls-dest-rule.yaml
sleep 0.5
k2 exec -n vl3-test deployments/ubuntu-deployment -c ubuntu -- tcpdump -f '!icmp' -i nsm-1 -U -w - >dump-hosts-2-curl-mtls.pcap &
sleep 0.5
k2 -n vl3-test exec deployments/ubuntu-deployment -c ubuntu -- curl helloworld.my-vl3-network:5000/hello -sS
sleep 1
kill -2 $!

serviceentry.networking.istio.io/se-hw1 created
destinationrule.networking.istio.io/my-vl3-network created
[1] 44800
tcpdump: listening on nsm-1, link-type RAW (Raw IP), snapshot length 262144 bytes
Hello version: v1, instance: helloworld-v1-5c59cf7d49-cgqv7
[1]  + 44800 interrupt  kubectl --kubeconfig /Users/daniluzlov/kind-configs/config-2 exec -n vl3-test                                      



$ tshark -r dump-standard-curl-http.pcap | grep 'GET /hello'                                                                    [20:37:31]
   20   0.053723   172.16.0.5 → 172.16.0.3   HTTP 871 GET /hello HTTP/1.1 

$ ! tshark -r dump-standard-curl-mtls.pcap | grep HTTP                                                                          [20:37:38]

$ tshark -r dump-standard-curl-mtls.pcap | grep TCP                                                                             [20:38:08]
   15   0.022380   172.16.0.5 → 172.16.0.3   TCP 60 41270 → 5000 [SYN] Seq=0 Win=64400 Len=0 MSS=1400 SACK_PERM TSval=1007096694 TSecr=0 WS=128
   16   0.023018   172.16.0.3 → 172.16.0.5   TCP 60 5000 → 41270 [SYN, ACK] Seq=0 Ack=1 Win=65236 Len=0 MSS=1400 SACK_PERM TSval=2560835189 TSecr=1007096694 WS=128
   17   0.023043   172.16.0.5 → 172.16.0.3   TCP 52 41270 → 5000 [ACK] Seq=1 Ack=1 Win=64512 Len=0 TSval=1007096695 TSecr=2560835189
   19   0.023504   172.16.0.3 → 172.16.0.5   TCP 52 5000 → 41270 [ACK] Seq=1 Ack=518 Win=64768 Len=0 TSval=2560835189 TSecr=1007096695
   21   0.026664   172.16.0.5 → 172.16.0.3   TCP 52 41270 → 5000 [ACK] Seq=518 Ack=1389 Win=64256 Len=0 TSval=1007096698 TSecr=2560835191
   22   0.026674   172.16.0.3 → 172.16.0.5   TCP 828 5000 → 41270 [PSH, ACK] Seq=1389 Ack=518 Win=64768 Len=776 TSval=2560835191 TSecr=1007096695
   23   0.026680   172.16.0.5 → 172.16.0.3   TCP 52 41270 → 5000 [ACK] Seq=518 Ack=2165 Win=63488 Len=0 TSval=1007096698 TSecr=2560835191
   24   0.029146   172.16.0.3 → 172.16.0.5   TCP 828 [TCP Spurious Retransmission] 5000 → 41270 [PSH, ACK] Seq=1389 Ack=518 Win=64768 Len=776 TSval=2560835195 TSecr=1007096695
   25   0.029159   172.16.0.5 → 172.16.0.3   TCP 64 [TCP Window Update] 41270 → 5000 [ACK] Seq=518 Ack=2165 Win=64256 Len=0 TSval=1007096701 TSecr=2560835195 SLE=1389 SRE=2165
   27   0.029641   172.16.0.5 → 172.16.0.3   TCP 622 41270 → 5000 [PSH, ACK] Seq=1906 Ack=2165 Win=64256 Len=570 TSval=1007096701 TSecr=2560835195
   29   0.030322   172.16.0.3 → 172.16.0.5   TCP 52 5000 → 41270 [ACK] Seq=2165 Ack=1906 Win=64256 Len=0 TSval=2560835196 TSecr=1007096701
   30   0.030331   172.16.0.3 → 172.16.0.5   TCP 52 5000 → 41270 [ACK] Seq=2165 Ack=2476 Win=63744 Len=0 TSval=2560835196 TSecr=1007096701
   31   0.030335   172.16.0.3 → 172.16.0.5   TCP 52 5000 → 41270 [ACK] Seq=2165 Ack=3380 Win=62848 Len=0 TSval=2560835196 TSecr=1007096702
   33   0.193098   172.16.0.5 → 172.16.0.3   TCP 52 41270 → 5000 [ACK] Seq=3380 Ack=3553 Win=64256 Len=0 TSval=1007096865 TSecr=2560835358
   34   0.193123   172.16.0.3 → 172.16.0.5   TCP 1440 5000 → 41270 [ACK] Seq=3553 Ack=3380 Win=64256 Len=1388 TSval=2560835358 TSecr=1007096702
   35   0.193127   172.16.0.5 → 172.16.0.3   TCP 52 41270 → 5000 [ACK] Seq=3380 Ack=4941 Win=63232 Len=0 TSval=1007096865 TSecr=2560835358
   36   0.193130   172.16.0.3 → 172.16.0.5   TCP 1440 5000 → 41270 [ACK] Seq=4941 Ack=3380 Win=64256 Len=1388 TSval=2560835358 TSecr=1007096702
   37   0.193133   172.16.0.5 → 172.16.0.3   TCP 52 41270 → 5000 [ACK] Seq=3380 Ack=6329 Win=62080 Len=0 TSval=1007096865 TSecr=2560835358
   38   0.193134   172.16.0.3 → 172.16.0.5   TCP 812 5000 → 41270 [PSH, ACK] Seq=6329 Ack=3380 Win=64256 Len=760 TSval=2560835358 TSecr=1007096702
   39   0.193141   172.16.0.5 → 172.16.0.3   TCP 52 41270 → 5000 [ACK] Seq=3380 Ack=7089 Win=61440 Len=0 TSval=1007096865 TSecr=2560835358



$ k1 cluster-info dump --output-directory run-sample --output yaml --all-namespaces                                             [20:38:26]
Cluster info dumped to run-sample

```


