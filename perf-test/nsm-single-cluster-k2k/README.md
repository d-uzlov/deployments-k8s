## Requires

- [Load balancer](../loadbalancer)
- [Interdomain DNS](../dns)
- Interdomain spire
    - [Spire on first cluster](../../spire/cluster1)
    - [Spire on second cluster](../../spire/cluster2)
    - [Spiffe Federation](../spiffe_federation)
- [Interdomain nsm](../nsm)

## Run

Start nse
```bash
k1 create ns msm-perf-test-nsm-k2k
k1 apply -k ./perf-test/nsm-single-cluster-k2k/apps/nginx
k1 apply -f ./perf-test/nsm-single-cluster-k2k/apps/nginx/nginx-svc.yaml -n msm-perf-test-nsm-k2k
sleep 1
k1 -n msm-perf-test-nsm-k2k wait --for=condition=ready --timeout=5m pod -l app=nse-kernel
```

```bash
k1 create ns msm-perf-test-nsm-k2k
k1 apply -n msm-perf-test-nsm-k2k -f ./perf-test/nsm-single-cluster-k2k/apps/fortio.yaml

sleep 1
k1 -n msm-perf-test-nsm-k2k wait --for=condition=ready --timeout=5m pod -l app=fortio
```

Run test:
```bash
k1 -n msm-perf-test-nsm-k2k port-forward svc/fortio-service 8080:8080 &
sleep 1
# you can open http://127.0.0.1:8080/fortio/ to ensure that server is working

TEST_NAME=packet-02-09-nsm-k2k
rm -rf ./perf-test/nsm-single-cluster-k2k/results/raw-$TEST_NAME
mkdir -p ./perf-test/nsm-single-cluster-k2k/results/raw-$TEST_NAME
for i in {1..3}
do
    echo round $i
    curl -s -d @./perf-test/nsm-single-cluster-k2k/configs-fortio/sample.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm-single-cluster-k2k/results/raw-$TEST_NAME/sample-localhost-qps40k-t20-$i.json
    curl -s -d @./perf-test/nsm-single-cluster-k2k/configs-fortio/nginx-40k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm-single-cluster-k2k/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps40k-t1-$i.json
    curl -s -d @./perf-test/nsm-single-cluster-k2k/configs-fortio/nginx-k8s-40k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm-single-cluster-k2k/results/raw-$TEST_NAME/$TEST_NAME-nginx-k8s-qps40k-t1-$i.json
    curl -s -d @./perf-test/nsm-single-cluster-k2k/configs-fortio/nginx-100-1.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm-single-cluster-k2k/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps100-t1-$i.json
    curl -s -d @./perf-test/nsm-single-cluster-k2k/configs-fortio/nginx-1k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/nsm-single-cluster-k2k/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps1k-t1-$i.json
done
pkill -f "port-forward"
```

## Cleanup

```bash
pkill -f "port-forward"
k1 delete -k ./perf-test/nsm-single-cluster-k2k/apps/nginx
kubectl --kubeconfig=$KUBECONFIG1 delete ns msm-perf-test-nsm-k2k
```
