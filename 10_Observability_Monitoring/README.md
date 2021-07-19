# 10. Observability

Will demonstrate how to collect __telemetry__ information from the mesh.


# 10.1 Access Istio Envoy Proxy Log

This is enabled for profile `demo`
```sh
meshConfig:
    accessLogFile: /dev/stdout
```

First access `/productpage` through ingress gateway (AWS ELB)
```sh
for i in {1..5}; do curl $(echo $(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/productpage) -L -k; done
```

Get log from istio sidecar proxy by selecting a container name by `-c istio-proxy`
```sh
kubectl logs $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') -c istio-proxy | tail

# output
[2020-08-09T06:33:04.054Z] "GET /productpage HTTP/1.1" 200 - "-" "-" 0 5179 21 21 "192.168.34.210" "curl/7.54.0" "b00a9a1f-8f01-9b76-b47f-ca8515f15098" "a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com" "127.0.0.1:9080" inbound|9080|http|productpage.default.svc.cluster.local 127.0.0.1:58640 192.168.44.36:9080 192.168.34.210:0 outbound_.9080_._.productpage.default.svc.cluster.local default # <----- inbound to productpage: "inbound|9080|http|productpage.default.svc.cluster.local"

[2020-08-09T06:33:04.064Z] "GET /reviews/0 HTTP/1.1" 200 - "-" "-" 0 375 8 8 "-" "curl/7.54.0" "b00a9a1f-8f01-9b76-b47f-ca8515f15098" "reviews:9080" "192.168.65.133:9080" outbound|9080|v3|reviews.default.svc.cluster.local 192.168.44.36:53640 10.100.5.108:9080 192.168.44.36:37054 - - # <----- outbound from productpage to reviews: "outbound|9080|v3|reviews.default.svc.cluster.local"
```

# 10.2 Distributed Tracing with Jaeger Dashboard
Ref: https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/#generating-traces-using-the-bookinfo-sample

`Jaeger` is already installed when installing istio using `demo` profile.

The number of requests you see on `jaeger` depends on Istio’s sampling rate. You set this rate when you install Istio. The default sampling rate is 1%.

Open the dashboard
```
istioctl dashboard jaeger
```

![alt text](../imgs/jaeger_1.png "")

Click on the most recent trace at the top to see the details corresponding to the latest request to the `/productpage`.

![alt text](../imgs/jaeger_2.png "")

> The trace is comprised of a set of spans, where each span corresponds to a Bookinfo service, invoked during the execution of a /productpage request, or internal Istio component, for example: istio-ingressgateway.


# 10.3 Visualize Service Mesh using Kiali
Check Kiali dashboard
```sh
istioctl dashboard kiali
```

Username and password are both `admin` by default.

![alt text](../imgs/kiali.png "Kiali")

Check Grafana dashboard
Ref: https://istio.io/docs/tasks/observability/metrics/using-istio-dashboard/
```sh
istioctl dashboard grafana
```
![alt text](../imgs/grafana.png "Grafana")



