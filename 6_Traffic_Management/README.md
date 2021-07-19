# 6. Traffic Management

Remember in chapter 1, we talked about Istio Service Mesh Architecture:
- Data Plane (Envoy proxy)
- Control Plane (Istiod)

All traffic that your mesh services send and receive (data plane traffic) is __proxied through Envoy__, making it easy to direct and control traffic around your mesh without making any changes to your services.

![alt text](../imgs/istio_architecture.svg "Istio Architecture")


To recap what Data Plane's Envoy Proxy is capable of:
- Envoy Proxy: a high-performance proxy developed in C++ to mediate all inbound and outbound traffic for all services in the service mesh. Envoy proxies are the only Istio components that interact with data plane traffic.
    - Traffic control 
        - a different load balancing policy to traffic for a particular subset of service instances
        - Staged rollouts with %-based traffic split
        - HTTP/2 and gRPC proxies
    - Network resiliency
        - Timeouts
        - Retries
        - Circuit breakers
        - Fault injection
    - Security and Authentication
        - rate limiting
        - TLS termination
- __Istio Resources that enable above features__
    - __Virtual services__   # < ----- what is this?
        - without VirtualService, Envoy distributes traffic using round-robin load balancing between all service instances. This is what __K8s service's L4 load balancing__ can do for you.
    - __Destination rules__
    - __Gateways__
    - Service entries
        - if the destination’s host is outside Istio service mesh (e.g. AWS RDS endpoint), __non-mesh service needs to be added using a service entry__
    - Sidecars


# 6.1 Traffic Splitting (i.e. Canary, weight-based routing) using Virtual Service Subset

![alt text](../imgs/istio_destination_rule_traffic_splitting.png "")

## Step 1: Add version label to pods

![alt text](../imgs/istio_destination_rule_traffic_splitting2.png "")

```sh
# check pod labels for pod with app=reviews
kubectl get pod --show-labels --selector app=reviews

# output
NAME                          READY   STATUS    RESTARTS   AGE   LABELS
reviews-v1-7f6558b974-wtmj8   2/2     Running   0          23h   app=reviews,istio.io/rev=default,pod-template-hash=7f6558b974,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=reviews,service.istio.io/canonical-revision=v1,version=v1
reviews-v2-6cb6ccd848-hrpbg   2/2     Running   0          23h   app=reviews,istio.io/rev=default,pod-template-hash=6cb6ccd848,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=reviews,service.istio.io/canonical-revision=v2,version=v2
reviews-v3-cc56b578-dg7gq     2/2     Running   0          23h   app=reviews,istio.io/rev=default,pod-template-hash=cc56b578,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=reviews,service.istio.io/canonical-revision=v3,version=v3
```

Notice three different `reviews` pods have `version=v1`, `version=v2`, and `version=v3`. These pods are created from three separate deployments.

```sh
kubectl get deploy --selector app=reviews

# output
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
reviews-v1   1/1     1            1           23h
reviews-v2   1/1     1            1           23h
reviews-v3   1/1     1            1           23h
```

You can also see this on `Kiali` dashboard
```sh
# open kiali dashboard
istioctl dashboard kiali
```
![alt text](../imgs/kiali_reviews_workload.png "")



However, they are proxied by one service using `app=reviews` label
```sh
# show reviews service and its labels
kubectl get svc reviews --show-labels

# output
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE   LABELS
reviews   ClusterIP   10.100.5.108   <none>        9080/TCP   23h   app=reviews,service=reviews
```

![alt text](../imgs/kiali_reviews_service.png "")



## Step 2: Create Istio DestinationRule resource

![alt text](../imgs/istio_destination_rule_traffic_splitting3.png "")

A subset/version of a route destination is identified with a reference to a named service subset which must be declared in a corresponding DestinationRule.


### DestinationRule Anatomy
In [destination_rules_versioning.yaml](destination_rules_versioning.yaml),
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews # name of a service from the service registry (in-cluster or could be external)
  trafficPolicy: # service-level traffic policy
      loadBalancer:
        simple: ROUND_ROBIN # default, so you don't need to specify it explicitly
  subsets: # One or more named sets that represent individual versions of a service. Traffic policies can be overridden at subset level.
  - name: v1 # <--- named version/subset
    labels:
      version: v1 # label attached to Pod definition
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

Breakdown:
- host
    - name of a service from the service registry. 
    - __in-cluster__ service: looked up from Kubernetes services
    - __external__ (from K8s) service: looked up from the hosts declared by __ServiceEntries__
- trafficPolicy (more in details in coming sections)
    - load balancing policy
        - round robin, random, least connection
    - connection pool sizes
        - controlling the volume of connections to an upstream service
    - outlier detection
        - controlling eviction of unhealthy hosts from the load balancing pool
- subset
    - One or more __named sets that represent individual versions of a service__
    - trafficPolicy
        - Traffic policies can be overridden at subset level.


Apply 
```sh
kubectl apply -f destination_rules_versioning.yaml

# check
kubectl get dr
NAME      HOST      AGE
reviews   reviews   5s
```

![alt text](../imgs/kiali_reviews_destinationrule.png "")




## Step 3: Add Subset and Weight to route destination in Virtual Service

![alt text](../imgs/istio_destination_rule_traffic_splitting4.png "")

In [virtualservice_reviews_canary.yaml](virtualservice_reviews_canary.yaml),
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - reviews
  gateways: # names of gateways and sidecars that should apply these routes
  - bookinfo-gateway # Don't ONLY USE this gateway as "reviews" k8s service is used internally by productpage service, so this VS rule should be applied to Envoy sidecar proxy inside reviews pod, not edge proxy in gateway pod. 
  - mesh # applies to all the sidecars in the mesh. The reserved word mesh is used to imply all the sidecars in the mesh. When gateway field is omitted, the default gateway (mesh) will be used, which would apply the rule to all sidecars in the mesh. If a list of gateway names is provided, the rules will apply only to the gateways. To apply the rules to both gateways and sidecars, specify mesh as one of the gateway names. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 10 # <--- canary release. % of traffic to subset v1
    - destination:
        host: reviews
        subset: v2
      weight: 10
    - destination:
        host: reviews
        subset: v3
      weight: 80
```

Apply 
```sh
kubectl apply -f virtualservice_reviews_canary.yaml

# check
kubectl get vs
NAME      HOST      AGE
reviews   reviews   5s
```

![alt text](../imgs/kiali_reviews_virtualservice.png "")



## Step 4: Test and Verify Canary Traffic Splitting
Go to browser, now you should see 10% of time reviews is v1 (no stars) and v2 (black stars) respectively, and 80% of time v3 (red stars)

```sh
echo $(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/productpage

# check this from browser
a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com/productpage

# or make arbitrary # of requests from curl
for i in {1..20}; do curl $(echo $(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/productpage); done
```


```sh
# open kiali dashboard
istioctl dashboard kiali
```
Go to `kiali` dashboard -> Graph -> select `default` namespace > select Display dropdown and check off "Traffic Animation"
![alt text](../imgs/kiali_reviews_graph_settings.png "")

Then dashboard will show request percentage
![alt text](../imgs/kiali_reviews_graph_canary.png "")

You can also see request tracing/metrics from Service > Reviews > Traces
![alt text](../imgs/kiali_reviews_service_tracing.png "")



# 6.2 Identity-Based Routing
Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest

![alt text](../imgs/istio_ideintity_based_routing.png "")

In [virtualservice_reviews_header_condition_identity_based.yaml](virtualservice_reviews_header_condition_identity_based.yaml),
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - reviews
  gateways: # names of gateways and sidecars that should apply these routes
  - bookinfo-gateway # Don't ONLY USE this gateway as "reviews" k8s service is used internally by productpage service, so this VS rule should be applied to Envoy sidecar proxy inside reviews pod, not edge proxy in gateway pod. 
  - mesh # applies to all the sidecars in the mesh. The reserved word mesh is used to imply all the sidecars in the mesh. When gateway field is omitted, the default gateway (mesh) will be used, which would apply the rule to all sidecars in the mesh. If a list of gateway names is provided, the rules will apply only to the gateways. To apply the rules to both gateways and sidecars, specify mesh as one of the gateway names. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - match: # <---- this routing to apply to all requests from the specifieduser
    - headers: # Note: The keys uri, scheme, method, and authority will be ignored.
        end-user: # WARNING: merely passing this key-value pair in curl won't work as productpage service won't propagate custom header attributes to review service! You need to login as a user Ref: https://stackoverflow.com/a/50878208/1528958
          exact: log-in-as-this-user
    route:
    - destination:
        host: reviews
        subset: v1 # then route traffic destined to reviews (defined in hosts above) to backend reviews service of version 1
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 10 # <--- canary release. % of traffic to subset v1
    - destination:
        host: reviews
        subset: v2
      weight: 10
    - destination:
        host: reviews
        subset: v3
      weight: 80
  # - match: # <---- DON'T DO THIS as rules are evaluated in order
  #   - headers:
  #       end-user:
  #         exact: tester
  #   route:
  #   - destination:
  #       host: reviews
  #       subset: v1
```

You can add match conditions to http
```yaml
- match:
   - headers:
       end-user:
         exact: tester
```

Apply
```
kubectl apply -f virtualservice_reviews_header_condition_identity_based.yaml 
```

## How to Test
#### Warning: passing custom header attributes using browser plugin or curl won't work, as productpage app won't propagate those attributes to reviews app. Usually these work but not with `bookinfo` app:

### WON'T WORK:
~~From a browser, install `ModHeader` chrome plugin so you can modify HTTP header~~
![alt text](../imgs/reviews_header_condition_chrome_modheader_plugin.png "")

~~And you will see reviews v1, which is just texts without stars.~~
![alt text](../imgs/reviews_header_condition_chrome_dev_tools.png "")


~~From curl, make requests with header attribute~~
```sh
for i in {1..20}; do curl --verbose \
    --header "user-agent: tester" \
    $(echo $(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/productpage); done
```


### WORKS:
You login to bookinfo page as an user.

![alt text](../imgs/reviews_header_condition_identity_based_login.png "")
![alt text](../imgs/kiali_reviews_header_condition_identity_based_service.png "")

Kiali shows traffic is routed to reviews v1
![alt text](../imgs/kiali_reviews_header_condition_identity_based_graph.png "")






# 6.3 Query String Condition Routing

![alt text](../imgs/istio_query_string_routing.png "")


In [virtualservice_reviews_header_condition_query_string.yaml](virtualservice_reviews_header_condition_query_string.yaml),
```yaml
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - match:
    - queryParams: # <----- query parameter
        test-v2:
          exact: "true" # if "?test-v2=true" in query string. NOTE: prefix not supported for queryString
    route:
    - destination:
        host: reviews
        subset: v2 # then route traffic destined to reviews (defined in hosts above) to backend reviews service of version ２
```

Unfortunately, `productpage` app won't pass along header info including query string parameters to `reviews` app, so this can't be tested


# 6.4 URI (HTTP path) Condition Routing

![alt text](../imgs/istio_query_path_routing.png "")


In [virtualservice_reviews_header_condition_uri.yaml](virtualservice_reviews_header_condition_uri.yaml),
```yaml
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - match:
    - uri:
        regex: '^/productpage/v2' # could be exact|prefix|regex
      ignoreUriCase: true
    route:
    - destination:
        host: reviews
        subset: v2 # then route traffic destined to reviews (defined in hosts above) to backend reviews service of version ２
```

Unfortunately, `productpage` app won't pass along header info including query string parameters to `reviews` app, so this can't be tested




# 6.5 Inject Latency Delay for Resilience Testing using VirtualService
Refs: 
- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection-Delay
- https://istio.io/latest/docs/concepts/traffic-management/#fault-injection


![alt text](../imgs/istio_fault_delay.png "")


> Delays: Delays are timing failures. They mimic increased network latency or an overloaded upstream service

In [virtualservice_ratings_fault_injection_delay.yaml](virtualservice_ratings_fault_injection_delay.yaml),
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - ratings
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - match:
    - headers: # Note: The keys uri, scheme, method, and authority will be ignored.
        end-user: # WARNING: merely passing this key-value pair in curl won't work as productpage service won't propagate custom header attributes to review service! You need to login as a user Ref: https://stackoverflow.com/a/50878208/1528958
          exact: tester
    fault: 
      delay: # <------ inject latency 
        percentage:
          value: 100.0 # Percentage number in the range of [0.0, 100.0]. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#Percent
        fixedDelay: 10s # <----- delay
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```


Apply 
```
kubectl apply -f virtualservice_ratings_fault_injection_delay.yaml
```

Login as `tester` and access it from browser.


You expect the Bookinfo home page to load without errors in approximately __7-10 seconds__. However, the Reviews section displays an error message: `Sorry, product reviews are currently unavailable for this book.` 

This was the result of `productpage` receiving the timeout error from the `reviews` service, which couldn't get a response from `ratings` service which has 10s delay within `productpage` timeout (default application-level timeout of 3s times 1 retry = 7s).

```
productpage -> (3s timeout x 1 retry = max 6s) -> reviews -> ratings with 10s delay
```

![alt text](../imgs/ratings_fault_injection_delay.png "")


You can also check the delay from Developer Tools menu > Network tab > Reload the /productpage web page, and you see __7s__ latency:
![alt text](../imgs/ratings_fault_injection_delay_chrome_developer_tool.png "")



From Kiali
![alt text](../imgs/kiali_ratings_fault_injection_delay_graph.png "")

## Why 7 second latency despite 10s delay?
Ref: https://istio.io/latest/docs/tasks/traffic-management/fault-injection/#understanding-what-happened

> Timeout between the `reviews` and `ratings` service is __hard-coded at 10s__. However, there is also a hard-coded timeout between the `productpage` and the `reviews` service, coded as __3s + 1 retry for 6s__ total. As a result, the productpage call to reviews times out prematurely and throws an error after __6s__.



# 6.6 Return Arbitrary HTTP Response for Resilience Testing using VirtualService
Refs: 
- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection-Abort
- https://istio.io/latest/docs/concepts/traffic-management/#fault-injection


![alt text](../imgs/istio_fault_abort.png "")


Aborts: Aborts are crash failures. They mimic failures in upstream services. Aborts usually manifest in the form of HTTP error codes or TCP connection failures.

In [virtualservice_ratings_fault_injection_abort.yaml](virtualservice_ratings_fault_injection_abort.yaml),
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - ratings
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - match:
    - headers: # Note: The keys uri, scheme, method, and authority will be ignored.
        end-user: # WARNING: merely passing this key-value pair in curl won't work as productpage service won't propagate custom header attributes to review service! You need to login as a user Ref: https://stackoverflow.com/a/50878208/1528958
          exact: tester
    fault: 
      abort: # <----- return pre-specified HTTP code
        percentage:
          value: 100.0 # Percentage number in the range of [0.0, 100.0]. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#Percent
        httpStatus: 400
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```


Apply 
```
kubectl apply -f virtualservice_ratings_fault_injection_abort.yaml
```

Login as `tester` and access it from browser, the page loads __immediately__, as opposed to 7s delay like previous section, and the Ratings service is currently unavailable message appears. 
![alt text](../imgs/ratings_fault_injection_abort_chrome_developer_tool.png "")


Now you see red line from `reviews` pod to `ratings` service in Kiali.
![alt text](../imgs/kiali_virtualservice_ratings_fault_injection_abort_graph.png "")




# 6.7 Configure Timeout for Resilience Testing using VirtualService
Refs:
- https://istio.io/latest/docs/tasks/traffic-management/fault-injection/#injecting-an-http-abort-fault
- https://istio.io/latest/docs/concepts/traffic-management/#timeouts
- https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/


![alt text](../imgs/istio_timeout.png "")


A timeout is the amount of time that an Envoy proxy should wait for replies from a given service, ensuring that services don’t hang around waiting for replies indefinitely and that calls succeed or fail within a predictable timeframe. The __default timeout for HTTP requests is 15 seconds__, which means that if the service doesn’t respond within 15 seconds, the call fails.


Istio lets you easily adjust timeouts dynamically on a per-service basis using virtual services __without having to edit your service code__.

In [virtualservice_reviews_timeout.yaml](virtualservice_reviews_timeout.yaml),
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - reviews
  gateways: # names of gateways and sidecars that should apply these routes
  - bookinfo-gateway # Don't ONLY USE this gateway as "reviews" k8s service is used internally by productpage service, so this VS rule should be applied to Envoy sidecar proxy inside reviews pod, not edge proxy in gateway pod. 
  - mesh # applies to all the sidecars in the mesh. The reserved word mesh is used to imply all the sidecars in the mesh. When gateway field is omitted, the default gateway (mesh) will be used, which would apply the rule to all sidecars in the mesh. If a list of gateway names is provided, the rules will apply only to the gateways. To apply the rules to both gateways and sidecars, specify mesh as one of the gateway names. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - timeout: 1s # <--- default timeout for HTTP requests is 15 seconds, which means that if the service doesn’t respond within 15 seconds, the call fails
    route:
    - destination:
        host: reviews
        subset: v1
      weight: 10 # <--- canary release. % of traffic to subset v1
    - destination:
        host: reviews
        subset: v2
      weight: 10
    - destination:
        host: reviews
        subset: v3
      weight: 80
```

Apply
```
kubectl apply -f virtualservice_reviews_timeout.yaml
```

Visit the endpoint from browser.

The reason that the response takes 2 seconds, even though the istio timeout is configured at 1 second, is because there is a __hard-coded retry in the productpage service__, so it calls the timing out reviews service __twice__ before returning. 

Also, `productpage` __application-level timeout is hard-coded as 3s__. So if you try to override this value, it needs to be less than 3s, otherwise if it's more than 3s, stricter app-level timeout will take precedence, just for this `bookinfo` app.



Even if you change timeout to `10s`, app-level timeout of 3s will take precedence
```yaml
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - timeout: 10s
```


![alt text](../imgs/kiali_virtualservice_ratings_timeout_10s_graph.png "")




# 6.8 Configure Retry for Resilience Testing using VirtualService
Refs:
- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRetry
- https://istio.io/latest/docs/concepts/traffic-management/#retries
- https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-on


![alt text](../imgs/istio_retries.png "")


The following example configures a maximum of 3 retries to connect to this service subset after an initial call failure, each with a 3 second timeout.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - ratings
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - retries:
      attempts: 3 # default 1. The number of retries for a given request. The interval between retries will be determined automatically (25ms+). Actual number of retries attempted depends on the request timeout
      perTryTimeout: 3s
      retryOn: 5xx,gateway-error,reset,connect-failure,refused-stream,retriable-4xx # https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-on
    match:
    - headers: # Note: The keys uri, scheme, method, and authority will be ignored.
        end-user: # WARNING: merely passing this key-value pair in curl won't work as productpage service won't propagate custom header attributes to review service! You need to login as a user Ref: https://stackoverflow.com/a/50878208/1528958
          exact: tester
    fault: 
      delay: # inject latency only for "tester" user
        percentage:
          value: 100.0 # Percentage number in the range of [0.0, 100.0]. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#Percent
        fixedDelay: 15s
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

Even if we defined this retry of 3s timeout/retry and max 3 retries, taking 3x(1+3)=12s total, upstream `productpage` app-level default timeout is 3s with 1 retry (3x(1+1))=6s, so after 6s `productpage` will get timeout error from `reviews` service before getting response from `ratings`.


# 6.9 Mirror Live Traffic using VirtualService
Refs:
- https://istio.io/latest/docs/tasks/traffic-management/mirroring/
- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute


![alt text](../imgs/istio_mirror.png "")


Mirroring sends a copy of live traffic to a mirrored service.

When traffic gets mirrored, the requests are sent to the mirrored service with their __Host/Authority headers appended with -shadow__. For example, cluster-1 becomes cluster-1-shadow.

In [virtualservice_reviews_mirror.yaml](virtualservice_reviews_mirror.yaml),
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - reviews
  gateways: # names of gateways and sidecars that should apply these routes
  - bookinfo-gateway # Don't ONLY USE this gateway as "reviews" k8s service is used internally by productpage service, so this VS rule should be applied to Envoy sidecar proxy inside reviews pod, not edge proxy in gateway pod. 
  - mesh # applies to all the sidecars in the mesh. The reserved word mesh is used to imply all the sidecars in the mesh. When gateway field is omitted, the default gateway (mesh) will be used, which would apply the rule to all sidecars in the mesh. If a list of gateway names is provided, the rules will apply only to the gateways. To apply the rules to both gateways and sidecars, specify mesh as one of the gateway names. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - mirror: # <---- mirror traffic going to v3 to v1 as well, 100% of it
      host: reviews
      subset: v1
    mirror_percent: 100.0 # use double. https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute
    route:
    - destination:
        host: reviews
        subset: v3
      weight: 100
```

Apply
```sh
# apply normal ratings virtual service without delay nor abort
kubectl apply -f virtualservice_ratings.yaml 

kubectl apply -f virtualservice_reviews_mirror.yaml 
```

Visit the URL from browser

You can see traffic to v3:
![alt text](../imgs/kiali_virtualservice_reviews_mirror_v3_traffic.png "")

v1 is mirroring the traffic:
![alt text](../imgs/kiali_virtualservice_reviews_mirror_v1_traffic.png "")

And v1's istio proxy log shows __header host__ is appended with `-shadow` as in `reviews-shadow`
![alt text](../imgs/kiali_virtualservice_reviews_mirror_v1_log.png "")


However, v2 isn't receiving any traffic.
![alt text](../imgs/kiali_virtualservice_reviews_mirror_v2_traffic.png "")






# 6.10 Configure Load Balancing Policy using Destination Rule
Refs:
- https://istio.io/latest/docs/concepts/traffic-management/#destination-rule-example
- https://istio.io/latest/docs/reference/config/networking/destination-rule/


![alt text](../imgs/istio_destination_rule_lb_rules.png "")


> Destination Rule defines policies that apply to traffic intended for a service after routing has occurred

Destination Rule can configure:
- load balancing algorithm (random, least connection, round robin)
- connection pool size from the sidecar
- outlier detection settings to detect and evict unhealthy hosts from the load balancing pool

Without Istio DestinationRule's Traffic Policy, by default you are left with K8s service's L4 load balancing (round robin by default).

With Istio DestinationRule's Traffic Policy, you could control and split traffic to subset/versioned pods, and __among those pods you can configure fine-graied LB algorithms__ such as __round robin__, __random__, __least-connection__, __sticky sessions__, etc


## Round Robin Load Balancing (default)
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews # name of a service from the service registry
#   trafficPolicy: # service-level routing policy
#       loadBalancer:
#         simple: ROUND_ROBIN # default, so you don't need to specify it explicitly
```

## Random Load Balancing
It selects a random healthy host. The random load balancer generally performs better than round robin if no health checking policy is configured.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy: # service-level routing policy
      loadBalancer:
        simple: RANDOM # selects a random healthy host
```

## Least request load balancing
This uses an O(1) algorithm which selects two random healthy hosts and picks the host which has fewer active requests.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy: # service-level routing policy
      loadBalancer:
        simple: LEAST_CONN # selects two random healthy hosts and picks the host which has fewer active requests
```

## Multiple Load Balancing Rules for Subsets
In this example, we define:
- service-level trafficPolicy for `loadBalancer` and specify `ROUND_ROBIN`
- version specific trafficPolicy for `loadBalancer` and specify `LEAST_CONN` for subset `v1`
- `RANDOM` for subset `v2`
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews # name of a service from the service registry (in-cluster or could be external)
  trafficPolicy: # service-level traffic policy
    loadBalancer:
      simple: ROUND_ROBIN # default, so you don't need to specify it explicitly
  subsets: # One or more named sets that represent individual versions of a service. Traffic policies can be overridden at subset level.
  - name: v1
    labels:
      version: v1 # label attached to Pod definition
    trafficPolicy: # Version specific policies, which overrides service-level traffic 
      loadBalancer:
        simple: LEAST_CONN
  - name: v2
    labels:
      version: v2
    trafficPolicy: # Version specific policies, which overrides service-level traffic policy
      loadBalancer:
        simple: RANDOM
```


# 6.11 Enable Sticky Session for Load Balancing in DestinationRule
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews # name of a service from the service registry
  trafficPolicy: # service-level routing policy
    loadBalancer:
        consistentHash: # <--- changed from "simple"
            httpCookie:
                name: user # Name of the cookie.
                ttl: 1800s # Lifetime of the cookie
```


# 6.12 Enable Rate Limiting by Configuring Connection Pool sizes in DestinationRule
Connection pool settings can be applied at the TCP level as well as at HTTP level.. 

For example, the following yaml sets a limit of 100 connections to reviews service with a connect timeout of 30ms.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews # name of a service from the service registry
  trafficPolicy: # service-level routing policy
      connectionPool: # <------  ref: https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings
        http:
          http2MaxRequests: 1 # limit of 1 concurrent HTTP2 requests
          maxRequestsPerConnection: 1
        tcp:
          maxConnections: 2 # connection pool size of 2 HTTP1 connections
          connectTimeout: 30ms
          tcpKeepalive:
            time: 7200s
            interval: 75s
```



# 6.13 Enable Circuit Breaker by Configuring OutlierDetection in DestinationRule
Refs:
- https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/
- https://istio.io/latest/docs/concepts/traffic-management/#circuit-breakers

- HTTP services
    - hosts that continually return 5xx errors for API calls are ejected from the pool for a pre-defined period of time
- TCP services
    - connection __timeouts__ or connection __failures__ to a given host counts as an error when measuring the consecutive errors metric


In this yaml example, configures upstream hosts to be __scanned every 1 second__ so that any host that __fails 1 consecutive time__ with a 502, 503, or 504 error code will be __ejected for 3 minutes__.

In [destination_rules_productpage_circuit_breaker.yaml](destination_rules_productpage_circuit_breaker.yaml),
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage # name of a service from the service registry
  trafficPolicy: # service-level routing policy
      connectionPool: # ref: https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings
        http:
          http2MaxRequests: 1 # limit of 1 concurrent HTTP2 requests
          maxRequestsPerConnection: 1
        tcp:
          maxConnections: 2 # connection pool size of 2 HTTP1 connections
          connectTimeout: 30ms
          tcpKeepalive:
            time: 7200s
            interval: 75s
      outlierDetection: # <-----  ref: https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection
        consecutiveErrors: 1
        interval: 1s # scans every 1s
        baseEjectionTime: 3m 
```

Apply 
```
kubectl apply -f destination_rules_productpage_circuit_breaker.yaml
```

You will see Circuit Breaker icon on `productpage` workload on Kiali:
![alt text](../imgs/kiali_destination_rules_productpage_circuit_breaker_graph.png "")




## How to Test Circuit Breaker

Create a fortio pod
```sh
# create a pod
kubectl apply -f ../istio-1.6.7/samples/httpbin/sample-client/fortio-deploy.yaml

# test curl the bookinfo productpage URL
FORTIO_POD=$(kubectl get pods -lapp=fortio -o 'jsonpath={.items[0].metadata.name}')
BOOKINFO_URL=$(echo $(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/productpage)

kubectl exec -it "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -curl "$BOOKINFO_URL"
```

Now make tons of request
```sh
# Call the service with two concurrent connections (-c 2) and send 20 requests (-n 20)
kubectl exec -it "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning "$BOOKINFO_URL"

# output
15:49:54 I logger.go:114> Log level is now 3 Warning (was 2 Info)
Fortio 1.6.5 running at 0 queries per second, 2->2 procs, for 20 calls: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com/productpage
15:49:54 W http_client.go:143> Assuming http:// on missing scheme for 'a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com/productpage'
Starting at max qps with 2 thread(s) [gomax 2] for exactly 20 calls (10 per thread + 0)
Ended after 624.901397ms : 20 calls. qps=32.005
Aggregated Function Time : count 20 avg 0.060465896 +/- 0.01994 min 0.025406513 max 0.086700868 sum 1.20931791
# range, mid point, percentile, count
>= 0.0254065 <= 0.03 , 0.0277033 , 15.00, 3
> 0.03 <= 0.035 , 0.0325 , 25.00, 2
> 0.05 <= 0.06 , 0.055 , 30.00, 1
> 0.06 <= 0.07 , 0.065 , 65.00, 7
> 0.07 <= 0.08 , 0.075 , 90.00, 5
> 0.08 <= 0.0867009 , 0.0833504 , 100.00, 2
# target 50% 0.035
# target 75% 0.07
# target 90% 0.08
# target 99% 0.0927821
# target 99.9% 0.0934081
Sockets used: 11 (for perfect keepalive, would be 2)
Jitter: false
Code 200 : 10 (50.0 %)
Code 503 : 10 (50.0 %) # <------- half returning 5xx
Response Header Sizes : count 20 avg 84 +/- 84 min 0 max 168 sum 1680
Response Body/Total Sizes : count 20 avg 2811.3 +/- 2536 min 275 max 5351 sum 56226
All done 20 calls (plus 0 warmup) 38.917 ms avg, 31.2 qps
```

These were trapped by circuit breaking
```sh
Code 200 : 10 (50.0 %)
Code 503 : 10 (50.0 %) # <------- half returning 5xx
```