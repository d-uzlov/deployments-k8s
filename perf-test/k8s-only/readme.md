
Deploy test apps:
```bash
k1 apply -f ./perf-test/k8s-only/configs-k8s/namespace.yaml
k1 apply -n msm-perf-test-k8s-only -f ./perf-test/k8s-only/configs-k8s/nginx.yaml
k1 apply -n msm-perf-test-k8s-only -f ./perf-test/k8s-only/configs-k8s/fortio.yaml
sleep 1
k1 wait -n msm-perf-test-k8s-only --for=condition=ready --timeout=1m pod -l app=nginx
k1 wait -n msm-perf-test-k8s-only --for=condition=ready --timeout=1m pod -l app=fortio
```

Run test:
```bash
k1 -n msm-perf-test-k8s-only port-forward svc/fortio-service 8080:8080 &
sleep 1
# you can open http://127.0.0.1:8080/fortio/ to ensure that server is working

TEST_NAME=packet-02-09-k8s
rm -rf ./perf-test/k8s-only/results/raw-$TEST_NAME
mkdir -p ./perf-test/k8s-only/results/raw-$TEST_NAME
for i in {1..3}
do
    echo round $i
    curl -s -d @./perf-test/k8s-only/configs-fortio/sample.json "localhost:8080/fortio/rest/run" > ./perf-test/k8s-only/results/raw-$TEST_NAME/sample-localhost-qps40k-t20-$i.json
    curl -s -d @./perf-test/k8s-only/configs-fortio/nginx-40k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/k8s-only/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps40k-t1-$i.json
    curl -s -d @./perf-test/k8s-only/configs-fortio/nginx-100-1.json "localhost:8080/fortio/rest/run" > ./perf-test/k8s-only/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps100-t1-$i.json
    curl -s -d @./perf-test/k8s-only/configs-fortio/nginx-1k-1.json "localhost:8080/fortio/rest/run" > ./perf-test/k8s-only/results/raw-$TEST_NAME/$TEST_NAME-nginx-qps1k-t1-$i.json
done
pkill -f "port-forward"
```

Clear:
```bash
k1 delete -f ./perf-test/k8s-only/configs-k8s/namespace.yaml
```
