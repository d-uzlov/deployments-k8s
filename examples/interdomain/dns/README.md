## Setup DNS for two clusters

This example shows how to simply configure three k8s clusters to know each other. 
Can be skipped if clusters setupped with external DNS.

## Run

```bash
ip1=$(kubectl --kubeconfig=$KUBECONFIG1 get node -o go-template='{{ $addresses := (index .items 0).status.addresses }}{{range $addresses}}{{if eq .type "ExternalIP"}}{{.address}}{{break}}{{end}}{{end}}')
if [ -z "$ip1" ]; then
  ip1=$(kubectl --kubeconfig=$KUBECONFIG1 get node -o go-template='{{ $addresses := (index .items 0).status.addresses }}{{range $addresses}}{{if eq .type "InternalIP"}}{{.address}}{{break}}{{end}}{{end}}')
fi
ip2=$(kubectl --kubeconfig=$KUBECONFIG2 get node -o go-template='{{ $addresses := (index .items 0).status.addresses }}{{range $addresses}}{{if eq .type "ExternalIP"}}{{.address}}{{break}}{{end}}{{end}}')
if [ -z "$ip2" ]; then
  ip2=$(kubectl --kubeconfig=$KUBECONFIG2 get node -o go-template='{{ $addresses := (index .items 0).status.addresses }}{{range $addresses}}{{if eq .type "InternalIP"}}{{.address}}{{break}}{{end}}{{end}}')
fi
```

Add DNS forwarding from cluster1 to cluster2:
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        hosts /etc/coredns/customdomains.db my.cluster2. {
          fallthrough
        }
        file /etc/coredns/custom.db my.cluster2.
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        loop
        reload 5s
    }
  customdomains.db: |
    # 172.18.0.2 spire-server.spire.my.cluster2.
    # 172.18.0.2 nsmgr-proxy.nsm-system.my.cluster2.
    # 172.18.0.2 registry.nsm-system.my.cluster2.
  custom.db: |
    my.cluster2. IN SOA sns.dns.icann.org. noc.dns.icann.org. 2015082541 7200 3600 1209600 3600
    registry.nsm-system.my.cluster2. 86400 IN SRV 10 0 30002 raw.registry.nsm-system.my.cluster2.
    nsmgr-proxy.nsm-system.my.cluster2. 86400 IN SRV 10 0 30004 raw.nsmgr-proxy.nsm-system.my.cluster2.
    registry.nsm-system.my.cluster2.            IN      A       $ip2
    nsmgr-proxy.nsm-system.my.cluster2.            IN      A       $ip2
    spire-server.spire.my.cluster2.            IN      A       $ip2
EOF
```
```bash
curl https://raw.githubusercontent.com/d-uzlov/deployments-k8s/4cff7e4e8aa4d26982884b48a1cfc8b2cdf3bd23/examples/interdomain/dns/coredns.yaml | kubectl --kubeconfig=$KUBECONFIG1 -n kube-system patch deployments.apps coredns --patch-file /dev/stdin
```
```bash
kubectl --kubeconfig=$KUBECONFIG1 -n kube-system rollout restart deployment coredns &&
kubectl --kubeconfig=$KUBECONFIG1 -n kube-system rollout status deployment coredns
```

Add DNS forwarding from cluster2 to cluster1:
```bash
kubectl --kubeconfig=$KUBECONFIG2 apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        hosts /etc/coredns/customdomains.db my.cluster1. {
          fallthrough
        }
        file /etc/coredns/custom.db my.cluster1.
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        loop
        reload 5s
    }
  customdomains.db: |
    # 172.18.0.3 spire-server.spire.my.cluster1.
    # 172.18.0.3 nsmgr-proxy.nsm-system.my.cluster1.
    # 172.18.0.3 registry.nsm-system.my.cluster1.
  custom.db: |
    my.cluster1. IN SOA sns.dns.icann.org. noc.dns.icann.org. 2015082541 7200 3600 1209600 3600
    registry.nsm-system.my.cluster1. 86400 IN SRV 10 0 30002 raw.registry.nsm-system.my.cluster1.
    nsmgr-proxy.nsm-system.my.cluster1. 86400 IN SRV 10 0 30004 raw.nsmgr-proxy.nsm-system.my.cluster1.
    registry.nsm-system.my.cluster1.            IN      A       $ip1
    nsmgr-proxy.nsm-system.my.cluster1.            IN      A       $ip1
    spire-server.spire.my.cluster1.            IN      A       $ip1
EOF
```
```bash
curl https://raw.githubusercontent.com/d-uzlov/deployments-k8s/4cff7e4e8aa4d26982884b48a1cfc8b2cdf3bd23/examples/interdomain/dns/coredns.yaml | kubectl --kubeconfig=$KUBECONFIG2 -n kube-system patch deployments.apps coredns --patch-file /dev/stdin
```
```bash
kubectl --kubeconfig=$KUBECONFIG2 -n kube-system rollout restart deployment coredns &&
kubectl --kubeconfig=$KUBECONFIG2 -n kube-system rollout status deployment coredns
```

## Cleanup

