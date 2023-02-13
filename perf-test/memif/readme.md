
Deploy NSC and NSE:
```bash
k1 apply -k ./perf-test/memif/configs-k8s/nse-composition
sleep 1
kubectl wait --for=condition=ready --timeout=1m pod -l app=nse-kernel -n ns-nse-composition
kubectl wait --for=condition=ready --timeout=1m pod -l app=alpine -n ns-nse-composition
```

```bash
k1 apply -n ns-nse-composition -f ./perf-test/memif/configs-k8s/fortio.yaml
sleep 1
k1 -n ns-nse-composition wait --for=condition=ready --timeout=5m pod -l app=fortio
```

Run test:
```bash
k1 -n ns-nse-composition port-forward svc/fortio-service 8080:8080 &
sleep 1
# you can open http://127.0.0.1:8080/fortio/ to ensure that server is working

TEST_NAME=packet-02-10-memif
rm -rf ./perf-test/memif/results/raw-$TEST_NAME
mkdir -p ./perf-test/memif/results/raw-$TEST_NAME
for i in {1..10}
do
    echo round $i
    curl -s -d @./perf-test/memif/configs-fortio/sample.json "localhost:8080/fortio/rest/run" > ./perf-test/memif/results/raw-$TEST_NAME/$TEST_NAME-sample-localhost-qps40k-t20-$i.json
    curl -s -d @./perf-test/memif/configs-fortio/nginx-40k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/memif/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps40k-t1-$i.json
    curl -s -d @./perf-test/memif/configs-fortio/nginx-k8s-40k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/memif/results/raw-$TEST_NAME/$TEST_NAME-nginx-k8s-qps40k-t1-$i.json
    curl -s -d @./perf-test/memif/configs-fortio/nginx-100-1.json "localhost:8080/fortio/rest/run" > ./perf-test/memif/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps100-t1-$i.json
    curl -s -d @./perf-test/memif/configs-fortio/nginx-1k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/memif/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps1k-t1-$i.json
done
pkill -f "port-forward"
```

## Cleanup

Delete ns:
```bash
pkill -f "port-forward"
k1 delete ns ns-nse-composition
```
