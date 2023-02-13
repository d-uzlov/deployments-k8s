## Requires

- [Load balancer](../loadbalancer)
- [Interdomain DNS](../dns)
- Interdomain spire
    - [Spire on first cluster](../../spire/cluster1)
    - [Spire on second cluster](../../spire/cluster2)
    - [Spiffe Federation](../spiffe_federation)
- [Interdomain nsm](../nsm)

## Run

Start vl3
```bash
kubectl --kubeconfig=$KUBECONFIG1 apply -k ./perf-test/nsm/vl3-dns
sleep 1
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=5m pod -l app=vl3-ipam
kubectl --kubeconfig=$KUBECONFIG1 -n ns-dns-vl3 wait --for=condition=ready --timeout=5m pod -l app=nse-vl3-vpp
```

```bash
k1 create ns msm-perf-test-nsm
k1 apply -n msm-perf-test-nsm -f ./perf-test/nsm/apps/nginx.yaml

k2 create ns msm-perf-test-nsm
k2 apply -n msm-perf-test-nsm -f ./perf-test/nsm/apps/fortio.yaml

sleep 1
k1 -n msm-perf-test-nsm wait --for=condition=ready --timeout=5m pod -l app=nginx
k2 -n msm-perf-test-nsm wait --for=condition=ready --timeout=5m pod -l app=fortio
```

Run test:
```bash
k2 -n msm-perf-test-nsm port-forward svc/fortio-service 8080:8080 &
sleep 1
# you can open http://127.0.0.1:8080/fortio/ to ensure that server is working

TEST_NAME=packet-02-08-nsm
rm -rf ./perf-test/nsm/results/raw-$TEST_NAME
mkdir -p ./perf-test/nsm/results/raw-$TEST_NAME
for i in {1..3}
do
    echo round $i
    curl -s -d @./perf-test/nsm/configs-fortio/sample.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm/results/raw-$TEST_NAME/sample-localhost-qps40k-t20-$i.json
    curl -s -d @./perf-test/nsm/configs-fortio/nginx-40k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps40k-t1-$i.json
    curl -s -d @./perf-test/nsm/configs-fortio/nginx-100-1.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps100-t1-$i.json
    curl -s -d @./perf-test/nsm/configs-fortio/nginx-1k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps1k-t1-$i.json
done
pkill -f "port-forward"
```

## Cleanup

```bash
pkill -f "port-forward"
kubectl --kubeconfig=$KUBECONFIG1 delete -k ./perf-test/nsm/vl3-dns
kubectl --kubeconfig=$KUBECONFIG1 delete ns msm-perf-test-nsm
kubectl --kubeconfig=$KUBECONFIG2 delete ns msm-perf-test-nsm
```
