# 8 Authentication and Authorization using JWT 

# 8.1 Enable End-User Authentication with JWT Token for ENTIRE Gateway using RequestAuthentication and AuthorizationPolicy
Ref: 
- https://istio.io/latest/docs/tasks/security/authentication/authn-policy/#end-user-authentication
- https://istio.io/latest/docs/tasks/security/authentication/authn-policy/#require-a-valid-token
- https://istio.io/latest/docs/reference/config/security/authorization-policy/#AuthorizationPolicy
- https://istio.io/latest/docs/tasks/security/authorization/authz-jwt/


![alt text](../imgs/istio_requestauthentication_authorizationpolicy_jwt.png "")


## Step 1: Create Istio Request Authentication and Authorization policy

In [request_authentication.yaml](request_authentication.yaml), add a __request authentication policy__ that requires end-user JWT for the ingress gateway.
```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-example
  namespace: istio-system # apply the policy to the namespace of the workload it selects
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: testing@secure.istio.io
    jwksUri: https://raw.githubusercontent.com/istio/istio/release-1.6/security/tools/jwt/samples/jwks.json
```

In [authorization_policy.yaml](authorization_policy.yaml), add an __authorization policy__ with a rule specifying a DENY action for requests without __request principals__, which are available only when __valid JWT tokens__ are provided.
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY # reject requests with matching with rules below
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"] # <---- if request principals (valid JWT) not available in header. Ref: https://istio.io/latest/docs/tasks/security/authentication/authn-policy/#require-a-valid-token
```

Apply
```
kubectl apply -f request_authentication.yaml
kubectl apply -f authorization_policy.yaml
```

## Test JWT Authentication Working

Try accessing `/headers` endpoint without passing JWT token, it'll fail with `HTTP/1.1 403 Forbidden`:
```sh
curl $INGRESS_HOST/headers -v

# output
*   Trying 34.214.199.244...
* TCP_NODELAY set
* Connected to a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com (34.214.199.244) port 80 (#0)
> GET /headers HTTP/1.1
> Host: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 403 Forbidden # <---- error
< content-length: 19
< content-type: text/plain
< date: Sat, 08 Aug 2020 09:23:47 GMT
< server: istio-envoy
< 
* Connection #0 to host a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com left intact
```

Try with passing an __INVALID__ token `deadbeef` in `Authorization: Bearer` header, it'll return `HTTP 401 Unauthorized`
```sh
curl --header "Authorization: Bearer deadbeef" $INGRESS_HOST/headers -v

# output
*   Trying 54.149.143.27...
* TCP_NODELAY set
* Connected to a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com (54.149.143.27) port 80 (#0)
> GET /headers HTTP/1.1
> Host: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> Authorization: Bearer deadbeef
> 
< HTTP/1.1 401 Unauthorized
< content-length: 50
< content-type: text/plain
< date: Sat, 08 Aug 2020 09:17:49 GMT
< server: istio-envoy
< 
* Connection #0 to host a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com left intact
Jwt is not in the form of Header.Payload.Signature
```

Try with passing an __VALID__ token in `Authorization: Bearer` header, it'll succeed with `200`
```sh
# get a valid JWT token
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.6/security/tools/jwt/samples/demo.jwt -s)
echo $TOKEN

curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -v

# successful output
*   Trying 34.214.199.244...
* TCP_NODELAY set
* Connected to a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com (34.214.199.244) port 80 (#0)
> GET /headers HTTP/1.1
> Host: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.CfNnxWP2tcnR9q0vxyxweaF3ovQYHYZl82hAUsn21bwQd9zP7c-LS9qd_vpdLG4Tn1A15NxfCjp5f7QNBUo-KC9PJqYpgGbaXhaGx7bEdFWjcwv3nZzvc7M__ZpaCERdwU7igUmJqYGBYQ51vr2njU9ZimyKkfDe3axcyiBZde7G6dabliUosJvvKOPcKIWPccCgefSj_GNfwIip3-SsFdlR7BtbVUcqR-yv-XOxJ3Uc1MI0tz3uMiiZcyPV7sNCU4KRnemRIMHVOfuvHsU60_GhGbiSFzgPTAa9WTltbnarTbxudb_YEOx12JiwYToeX0DCPb43W1tzIBxgm8NxUg # <----- JWT token in header
> 
< HTTP/1.1 200 OK
< server: istio-envoy
< date: Sat, 08 Aug 2020 09:20:13 GMT
< content-type: application/json
< content-length: 1689
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 4
```


# 8.2 Enable End-User Authentication with JWT Token per Path


![alt text](../imgs/istio_requestauthentication_authorizationpolicy_jwt_path.png "")


For example, change the authorization policy to only require JWT on `/headers`.

In [authorization_policy_httpbin_path.yaml](authorization_policy_httpbin_path.yaml),
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-path
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY # reject requests with matching with rules below
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"] 
    to:
      - operation:
          paths: ["/headers"] # <---- deny if invalid JWT token for /headers 
```

Apply
```sh
# first delete global authorization policy applied
kubectl delete authorizationpolicy frontend-ingress -n istio-system

kubectl apply -f authorization_policy_httpbin_path.yaml
```

## Test /headers fail without a valid JWT

Without JWT fails
```sh
curl $INGRESS_HOST/headers -v

# expected failing output
*   Trying 54.149.143.27...
* TCP_NODELAY set
* Connected to a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com (54.149.143.27) port 80 (#0)
> GET /headers HTTP/1.1
> Host: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 403 Forbidden # <----- authorization failed
< content-length: 19
< content-type: text/plain
< date: Sat, 08 Aug 2020 09:41:09 GMT
< server: istio-envoy
< 
* Connection #0 to host a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com left intact
RBAC: access denied # <----- authorization failed
```

With JWT succeeds
```sh
curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -v -L -k

# successful output
*   Trying 54.149.143.27...
* TCP_NODELAY set
* Connected to a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com (54.149.143.27) port 80 (#0)
> GET /headers HTTP/1.1
> Host: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.CfNnxWP2tcnR9q0vxyxweaF3ovQYHYZl82hAUsn21bwQd9zP7c-LS9qd_vpdLG4Tn1A15NxfCjp5f7QNBUo-KC9PJqYpgGbaXhaGx7bEdFWjcwv3nZzvc7M__ZpaCERdwU7igUmJqYGBYQ51vr2njU9ZimyKkfDe3axcyiBZde7G6dabliUosJvvKOPcKIWPccCgefSj_GNfwIip3-SsFdlR7BtbVUcqR-yv-XOxJ3Uc1MI0tz3uMiiZcyPV7sNCU4KRnemRIMHVOfuvHsU60_GhGbiSFzgPTAa9WTltbnarTbxudb_YEOx12JiwYToeX0DCPb43W1tzIBxgm8NxUg
> 
< HTTP/1.1 200 OK
< server: istio-envoy
< date: Sat, 08 Aug 2020 09:40:57 GMT
< content-type: application/json
< content-length: 1689
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 3
```

## Test /ip succeed without a valid JWT
```sh
curl $INGRESS_HOST/ip -v -k -L

# successful output
*   Trying 54.149.143.27...
* TCP_NODELAY set
* Connected to a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com (54.149.143.27) port 80 (#0)
> GET /ip HTTP/1.1
> Host: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< server: istio-envoy
< date: Sat, 08 Aug 2020 09:41:17 GMT
< content-type: application/json
< content-length: 32
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 3
< 
{
  "origin": "192.168.69.89"
}
```


# 8.3  Enable End-User Authentication with JWT Token per Host And Path

Now we can narrow down JWT config for not only path `/headers` but also `*.sni.com` host/


In [authorization_policy_httpbin_path_host.yaml](authorization_policy_httpbin_path_host.yaml),
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-path-host
  namespace: istio-system # If set to root (i.e. "istio-sytem") namespace, the policy applies to all namespaces in a mesh.
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY # reject requests with matching with rules below
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"] # <---- if request principals (valid JWT) not available in header. Ref: https://istio.io/latest/docs/tasks/security/authentication/authn-policy/#require-a-valid-token
    to:
    - operation:
        # these are AND condition (i.e. test.sni.com/headers)
        paths: ["/headers"] # <---- deny if invalid JWT token for /headers
        hosts: ["test.sni.com"] 
    # this is OR condition (i.e. either */headers OR test.sni.com/*)
    # - operation:
    #     hosts: ["test.sni.com"] 
```

Apply
```sh
# first delete previous authorization policy
kubectl delete -f authorization_policy_httpbin_path.yaml

kubectl apply -f authorization_policy_httpbin_path_host.yaml
```


## Test "/headers" with "*.sni.com" fail without a valid JWT

```sh
curl -v \
  --resolve "test.sni.com:$SECURE_INGRESS_PORT:$(host $(echo $INGRESS_HOST) | tail -1 | awk '{ print $4 }')" \
  "https://test.sni.com:$SECURE_INGRESS_PORT/headers" -k

# expected failing output
* Added test.sni.com:443:54.149.143.27 to DNS cache
* Hostname test.sni.com was found in DNS cache
*   Trying 54.149.143.27...
* TCP_NODELAY set
* Connected to test.sni.com (54.149.143.27) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-CHACHA20-POLY1305
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.sni.com # <----- using right cert thanks to SNI
*  start date: Aug  7 08:12:52 2020 GMT
*  expire date: Aug  7 08:12:52 2021 GMT
*  issuer: CN=*.sni.com
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7fdfc600ba00)
> GET /headers HTTP/2
> Host: test.sni.com # <------ host is *.sni.com
> User-Agent: curl/7.54.0
> Accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 403 
< content-length: 19
< content-type: text/plain
< date: Sat, 08 Aug 2020 10:38:35 GMT
< server: istio-envoy
< 
* Connection #0 to host test.sni.com left intact
RBAC: access denie  # <----- authorization failed
```


## Test "/headers" with "*.sni.com" succeed with a valid JWT
```sh
# pass --header "Authorization: Bearer $TOKEN"
curl -v \
  --resolve "test.sni.com:$SECURE_INGRESS_PORT:$(host $(echo $INGRESS_HOST) | tail -1 | awk '{ print $4 }')" \
  --header "Authorization: Bearer $TOKEN" \
  "https://test.sni.com:$SECURE_INGRESS_PORT/headers" -k


# successful output
* Added test.sni.com:443:34.214.199.244 to DNS cache
* Hostname test.sni.com was found in DNS cache
*   Trying 34.214.199.244...
* TCP_NODELAY set
* Connected to test.sni.com (34.214.199.244) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-CHACHA20-POLY1305
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.sni.com # <----- using right cert thanks to SNI
*  start date: Aug  7 08:12:52 2020 GMT
*  expire date: Aug  7 08:12:52 2021 GMT
*  issuer: CN=*.sni.com
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7fe88400ba00)
> GET /headers HTTP/2
> Host: test.sni.com # <------ host is *.sni.com
> User-Agent: curl/7.54.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.CfNnxWP2tcnR9q0vxyxweaF3ovQYHYZl82hAUsn21bwQd9zP7c-LS9qd_vpdLG4Tn1A15NxfCjp5f7QNBUo-KC9PJqYpgGbaXhaGx7bEdFWjcwv3nZzvc7M__ZpaCERdwU7igUmJqYGBYQ51vr2njU9ZimyKkfDe3axcyiBZde7G6dabliUosJvvKOPcKIWPccCgefSj_GNfwIip3-SsFdlR7BtbVUcqR-yv-XOxJ3Uc1MI0tz3uMiiZcyPV7sNCU4KRnemRIMHVOfuvHsU60_GhGbiSFzgPTAa9WTltbnarTbxudb_YEOx12JiwYToeX0DCPb43W1tzIBxgm8NxUg
> 
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 200  # <----- HTTP 200
< server: istio-envoy
< date: Sat, 08 Aug 2020 10:43:49 GMT
< content-type: application/json
< content-length: 1631
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 3
```


## Test "/headers" with "*.elb.amazonaws.com" succeed WITHOUT a valid JWT
```sh
# this time, you don't pass --resolve for "*.sni.com", because we are sending request to "*.elb.amazonaws.com"
curl https://$INGRESS_HOST/headers -v -k  

# successful output
*Trying 54.149.143.27...* TCP_NODELAY set
* Connected to a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com (54.149.143.27) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-CHACHA20-POLY1305
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.elb.amazonaws.com # <------ host is NOT *.sni.com
*  start date: Aug  5 11:00:46 2020 GMT
*  expire date: Aug  5 11:00:46 2021 GMT
*  issuer: CN=*.elb.amazonaws.com
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7ff33f80ba00)
> GET /headers HTTP/2
> Host: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 200  # <------ HTTP 200
< server: istio-envoy
< date: Sat, 08 Aug 2020 10:46:01 GMT
< content-type: application/json
< content-length: 1689
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 3
```



# 8.4 Enable Authorization using HTTP Header Attribute
Ref: https://istio.io/latest/docs/tasks/security/authorization/authz-deny/#explicitly-deny-a-request

In [authorization_policy_deny_header.yaml](authorization_policy_deny_header.yaml),
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-header
  namespace: istio-system # <----- make sure namespace is root "istio-system" so this config applies to gateway
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY # reject requests with matching with rules below
  rules:
  - to:
    - operation:
        paths: ["/headers"] # <---- deny if invalid JWT token for /headers
        hosts: ["*.sni.com"] 
    when:
    - key: request.headers[x-token] 
      notValues: ["admin"] # <----- deny if x-token is not admin
```

Apply
```sh
# first delete previous authorization policy
kubectl delete -f authorization_policy_httpbin_path_host.yaml

kubectl apply -f authorization_policy_deny_header.yaml 
```

## Test "/headers" with "*.sni.com" with a valid JWT fail WITHOUT "x-token" header

```sh
# should fail because of missing "x-token" header for /headers endpoint
curl -v \
  --resolve "test.sni.com:$SECURE_INGRESS_PORT:$(host $(echo $INGRESS_HOST) | tail -1 | awk '{ print $4 }')" \
  --header "Authorization: Bearer $TOKEN" \
  "https://test.sni.com:$SECURE_INGRESS_PORT/headers" -k

# expected failing output
* Added test.sni.com:443:34.214.199.244 to DNS cache
* Hostname test.sni.com was found in DNS cache
*   Trying 34.214.199.244...
* TCP_NODELAY set
* Connected to test.sni.com (34.214.199.244) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-CHACHA20-POLY1305
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.sni.com
*  start date: Aug  7 08:12:52 2020 GMT
*  expire date: Aug  7 08:12:52 2021 GMT
*  issuer: CN=*.sni.com
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7fa2f500ba00)
> GET /headers HTTP/2
> Host: test.sni.com
> User-Agent: curl/7.54.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.CfNnxWP2tcnR9q0vxyxweaF3ovQYHYZl82hAUsn21bwQd9zP7c-LS9qd_vpdLG4Tn1A15NxfCjp5f7QNBUo-KC9PJqYpgGbaXhaGx7bEdFWjcwv3nZzvc7M__ZpaCERdwU7igUmJqYGBYQ51vr2njU9ZimyKkfDe3axcyiBZde7G6dabliUosJvvKOPcKIWPccCgefSj_GNfwIip3-SsFdlR7BtbVUcqR-yv-XOxJ3Uc1MI0tz3uMiiZcyPV7sNCU4KRnemRIMHVOfuvHsU60_GhGbiSFzgPTAa9WTltbnarTbxudb_YEOx12JiwYToeX0DCPb43W1tzIBxgm8NxUg
> 
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 403 
< content-length: 19
< content-type: text/plain
< date: Sat, 08 Aug 2020 12:42:24 GMT
< server: istio-envoy
< 
* Connection #0 to host test.sni.com left intact
RBAC: access denied
```

## Test "/headers" with "*.sni.com" with a valid JWT fail with "x-token: test" header

```sh
# should fail because "x-token" value is not "admin" for /headers endpoint
curl -v \
  --resolve "test.sni.com:$SECURE_INGRESS_PORT:$(host $(echo $INGRESS_HOST) | tail -1 | awk '{ print $4 }')" \
  --header "Authorization: Bearer $TOKEN" \
  --header "x-token: test" \
  "https://test.sni.com:$SECURE_INGRESS_PORT/headers" -k

# expected failing output
* Added test.sni.com:443:34.214.199.244 to DNS cache
* Hostname test.sni.com was found in DNS cache
*   Trying 34.214.199.244...
* TCP_NODELAY set
* Connected to test.sni.com (34.214.199.244) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-CHACHA20-POLY1305
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.sni.com
*  start date: Aug  7 08:12:52 2020 GMT
*  expire date: Aug  7 08:12:52 2021 GMT
*  issuer: CN=*.sni.com
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7ff51380ba00)
> GET /headers HTTP/2
> Host: test.sni.com
> User-Agent: curl/7.54.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.CfNnxWP2tcnR9q0vxyxweaF3ovQYHYZl82hAUsn21bwQd9zP7c-LS9qd_vpdLG4Tn1A15NxfCjp5f7QNBUo-KC9PJqYpgGbaXhaGx7bEdFWjcwv3nZzvc7M__ZpaCERdwU7igUmJqYGBYQ51vr2njU9ZimyKkfDe3axcyiBZde7G6dabliUosJvvKOPcKIWPccCgefSj_GNfwIip3-SsFdlR7BtbVUcqR-yv-XOxJ3Uc1MI0tz3uMiiZcyPV7sNCU4KRnemRIMHVOfuvHsU60_GhGbiSFzgPTAa9WTltbnarTbxudb_YEOx12JiwYToeX0DCPb43W1tzIBxgm8NxUg
> x-token: test # <------- admin is needed
> 
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 403 
< content-length: 19
< content-type: text/plain
< date: Sat, 08 Aug 2020 12:45:12 GMT
< server: istio-envoy
< 
* Connection #0 to host test.sni.com left intact
RBAC: access denied
```


## Test "/headers" with "*.sni.com" with a valid JWT succeed with "x-token" header

```sh
curl -v \
  --resolve "test.sni.com:$SECURE_INGRESS_PORT:$(host $(echo $INGRESS_HOST) | tail -1 | awk '{ print $4 }')" \
  --header "Authorization: Bearer $TOKEN" \
  --header "x-token: admin" \
  "https://test.sni.com:$SECURE_INGRESS_PORT/headers" -k

# expected successful output
* Added test.sni.com:443:34.214.199.244 to DNS cache
* Hostname test.sni.com was found in DNS cache
*   Trying 34.214.199.244...
* TCP_NODELAY set
* Connected to test.sni.com (34.214.199.244) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-CHACHA20-POLY1305
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.sni.com
*  start date: Aug  7 08:12:52 2020 GMT
*  expire date: Aug  7 08:12:52 2021 GMT
*  issuer: CN=*.sni.com
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7f913480ba00)
> GET /headers HTTP/2
> Host: test.sni.com
> User-Agent: curl/7.54.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.CfNnxWP2tcnR9q0vxyxweaF3ovQYHYZl82hAUsn21bwQd9zP7c-LS9qd_vpdLG4Tn1A15NxfCjp5f7QNBUo-KC9PJqYpgGbaXhaGx7bEdFWjcwv3nZzvc7M__ZpaCERdwU7igUmJqYGBYQ51vr2njU9ZimyKkfDe3axcyiBZde7G6dabliUosJvvKOPcKIWPccCgefSj_GNfwIip3-SsFdlR7BtbVUcqR-yv-XOxJ3Uc1MI0tz3uMiiZcyPV7sNCU4KRnemRIMHVOfuvHsU60_GhGbiSFzgPTAa9WTltbnarTbxudb_YEOx12JiwYToeX0DCPb43W1tzIBxgm8NxUg
> x-token: admin
> 
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 200 
< server: istio-envoy
< date: Sat, 08 Aug 2020 12:44:09 GMT
< content-type: application/json
< content-length: 1656
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 3
< 
{
  "headers": {
    "Accept": "*/*", 
    "Content-Length": "0", 
    "Host": "test.sni.com", 
    "User-Agent": "curl/7.54.0", 
    "X-B3-Sampled": "1", 
    "X-B3-Spanid": "b3ffaa414a45c70c", 
    "X-B3-Traceid": "0a31a764cba65071b3ffaa414a45c70c", 
    "X-Envoy-Decorator-Operation": "httpbin.default.svc.cluster.local:80/headers", 
    "X-Envoy-Internal": "true", 
    "X-Envoy-Peer-Metadata": "ChoKCkNMVVNURVJfSUQSDBoKS3ViZXJuZXRlcwofCgxJTlNUQU5DRV9JUFMSDxoNMTkyLjE2OC42Ni43NAqWAgoGTEFCRUxTEosCKogCCh0KA2FwcBIWGhRpc3Rpby1pbmdyZXNzZ2F0ZXdheQoTCgVjaGFydBIKGghnYXRld2F5cwoUCghoZXJpdGFnZRIIGgZUaWxsZXIKGQoFaXN0aW8SEBoOaW5ncmVzc2dhdGV3YXkKIQoRcG9kLXRlbXBsYXRlLWhhc2gSDBoKNWQ4NjlmNWJiZgoSCgdyZWxlYXNlEgcaBWlzdGlvCjkKH3NlcnZpY2UuaXN0aW8uaW8vY2Fub25pY2FsLW5hbWUSFhoUaXN0aW8taW5ncmVzc2dhdGV3YXkKLwojc2VydmljZS5pc3Rpby5pby9jYW5vbmljYWwtcmV2aXNpb24SCBoGbGF0ZXN0ChoKB01FU0hfSUQSDxoNY2x1c3Rlci5sb2NhbAovCgROQU1FEicaJWlzdGlvLWluZ3Jlc3NnYXRld2F5LTVkODY5ZjViYmYtYnZweHMKGwoJTkFNRVNQQUNFEg4aDGlzdGlvLXN5c3RlbQpdCgVPV05FUhJUGlJrdWJlcm5ldGVzOi8vYXBpcy9hcHBzL3YxL25hbWVzcGFjZXMvaXN0aW8tc3lzdGVtL2RlcGxveW1lbnRzL2lzdGlvLWluZ3Jlc3NnYXRld2F5CqcBChFQTEFURk9STV9NRVRBREFUQRKRASqOAQogCg5hd3NfYWNjb3VudF9pZBIOGgwxNjQ5MjU1OTYzMTUKJQoVYXdzX2F2YWlsYWJpbGl0eV96b25lEgwaCnVzLXdlc3QtMmQKKAoPYXdzX2luc3RhbmNlX2lkEhUaE2ktMDQ3Yzc0MTQ3NjZhZjIwNmUKGQoKYXdzX3JlZ2lvbhILGgl1cy13ZXN0LTIKOQoPU0VSVklDRV9BQ0NPVU5UEiYaJGlzdGlvLWluZ3Jlc3NnYXRld2F5LXNlcnZpY2UtYWNjb3VudAonCg1XT1JLTE9BRF9OQU1FEhYaFGlzdGlvLWluZ3Jlc3NnYXRld2F5", 
    "X-Envoy-Peer-Metadata-Id": "router~192.168.66.74~istio-ingressgateway-5d869f5bbf-bvpxs.istio-system~istio-system.svc.cluster.local", 
    "X-Token": "admin" # <------- admin is needed
  }
}
```


# 8.5 Enable End-User Authorization by Source IP
Refs: 
- https://istio.io/latest/docs/tasks/security/authorization/authz-ingress/#ip-based-allow-list-and-deny-list
- https://istio.io/latest/docs/reference/config/security/authorization-policy/

Here is the precedence order of DENY and ALLOW policies:
1. If there are any DENY policies that match the request, deny the request.
2. If there are no ALLOW policies for the workload, allow the request.
3. If any of the ALLOW policies match the request, allow the request.
4. Deny the request.


First get your own IP
```sh
CLIENT_IP=$(curl $INGRESS_HOST/ip -L -k -s | grep "origin" | cut -d'"' -f 4) && echo $CLIENT_IP

# output
192.168.69.89
```

```sh
# first delete previous authorization policy
kubectl delete -f authorization_policy_deny_header.yaml
```

Apply
```yaml
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-ip
  namespace: istio-system # If set to root (i.e. "istio-sytem") namespace, the policy applies to all namespaces in a mesh.
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: DENY
  rules:
  - from:
    - source:
       ipBlocks: ["$CLIENT_IP"] # <--- deny request from this source IP
EOF
```


Test making a request 
```sh
curl $INGRESS_HOST/ip -v -L -k

# expected failing output
*   Trying 54.149.143.27...
* TCP_NODELAY set
* Connected to a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com (54.149.143.27) port 80 (#0)
> GET /ip HTTP/1.1
> Host: a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 403 Forbidden
< content-length: 19
< content-type: text/plain
< date: Sat, 08 Aug 2020 11:30:28 GMT
< server: istio-envoy
< 
* Connection #0 to host a5a1acc36239d46038f3dd828465c946-706040707.us-west-2.elb.amazonaws.com left intact
RBAC: access denied # <----- authorization denied
```

```
kubectl delete -f authorization_policy_deny_ip.yaml
```



# 8.6 Check Authorization Policy for Pod
Check pod's authorization policy
```sh
istioctl exp authz check httpbin -n default
```

Output
```sh
hecked 17/41 listeners with node IP 10.1.103.9.
LISTENER[FilterChain]     CERTIFICATE          mTLS (MODE)     JWT (ISSUERS)     AuthZ (RULES)
0.0.0.0_80[0]             none                 no (none)       no (none)         no (none)
0.0.0.0_80[1]             none                 no (none)       no (none)         no (none)
0.0.0.0_3000[0]           none                 no (none)       no (none)         no (none)
0.0.0.0_3000[1]           none                 no (none)       no (none)         no (none)
0.0.0.0_8080[0]           none                 no (none)       no (none)         no (none)
0.0.0.0_8080[1]           none                 no (none)       no (none)         no (none)
0.0.0.0_8085[0]           none                 no (none)       no (none)         no (none)
0.0.0.0_8085[1]           none                 no (none)       no (none)         no (none)
0.0.0.0_9080[0]           none                 no (none)       no (none)         no (none)
0.0.0.0_9080[1]           none                 no (none)       no (none)         no (none)
0.0.0.0_9090[0]           none                 no (none)       no (none)         no (none)
0.0.0.0_9090[1]           none                 no (none)       no (none)         no (none)
0.0.0.0_9091[0]           none                 no (none)       no (none)         no (none)
0.0.0.0_9091[1]           none                 no (none)       no (none)         no (none)
0.0.0.0_9100[0]           none                 no (none)       no (none)         no (none)
0.0.0.0_9100[1]           none                 no (none)       no (none)         no (none)
0.0.0.0_9100[2]           none                 no (none)       no (none)         no (none)
0.0.0.0_9411[0]           none                 no (none)       no (none)         no (none)
0.0.0.0_9411[1]           none                 no (none)       no (none)         no (none)
0.0.0.0_14250[0]          none                 no (none)       no (none)         no (none)
0.0.0.0_14250[1]          none                 no (none)       no (none)         no (none)
0.0.0.0_14250[2]          none                 no (none)       no (none)         no (none)
virtualOutbound[0]        none                 no (none)       no (none)         no (none)
virtualOutbound[1]        none                 no (none)       no (none)         no (none)
virtualInbound[0]         noneSDS: default     yes (none)      no (none)         no (none)
virtualInbound[1]         none                 no (none)       no (none)         no (none)
virtualInbound[2]         none                 no (none)       no (none)         no (none)
virtualInbound[3]         noneSDS: default     yes (none)      no (none)         no (none)
virtualInbound[4]         none                 no (none)       no (none)         no (none)
0.0.0.0_15010[0]          none                 no (none)       no (none)         no (none)
0.0.0.0_15010[1]          none                 no (none)       no (none)         no (none)
0.0.0.0_15014[0]          none                 no (none)       no (none)         no (none)
0.0.0.0_15014[1]          none                 no (none)       no (none)         no (none)
0.0.0.0_20001[0]          none                 no (none)       no (none)         no (none)
0.0.0.0_20001[1]          none                 no (none)       no (none)         no (none)
10.1.103.9_9999[0]        noneSDS: default     yes (none)      no (none)         no (none)
10.1.103.9_9999[1]        none                 no (none)       no (none)         no (none)
10.1.103.9_15020          none                 no (none)       no (none)         no (none)
```


<!-- # 8.6 Authorize Access from Namespace

## BEFORE
`curl` in `non-istio` namespace can access `httpbin` in `default` namespace

Get a shell inside curl pod in `non-istio` namespace
```sh
# create a pod
kubectl apply -f ../ch7_Security_in_transit_TSL/pod_curl_non_istio.yaml

# curl httpbin in default namespace from curl coontainer in non-istio namespace
kubectl exec -it curl -n non-istio sh
curl httpbin.default/headers

# expected successful output 
{
  "headers": {
    "Accept": "*/*", 
    "Content-Length": "0", 
    "Host": "httpbin.default", 
    "User-Agent": "curl/7.71.1-DEV", 
    "X-B3-Sampled": "1", 
    "X-B3-Spanid": "8ad81217034f129c", 
    "X-B3-Traceid": "49473e3397096c2a8ad81217034f129c"
  }
}
```


## AFTER

In [authorization_policy_deny_namespace.yaml](authorization_policy_deny_namespace.yaml), it denies requests from the `non-istio` namespace to the `GET` and `POST` method on all workloads in the `default` namespace.
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: deny-namespace
 namespace: default
spec:
 action: DENY
 rules:
 - from:
   - source:
       namespaces: ["non-istio"]
   to:
   - operation:
       methods: ["GET","POST"]
```

Apply
```
kubectl apply -f authorization_policy_deny_namespace.yaml 
```

Get a shell inside curl pod in `non-istio` namespace
```sh
# curl httpbin in default namespace from curl coontainer in non-istio namespace
kubectl exec -it curl -n non-istio sh
curl httpbin.default/headers

# expected successful output 
{
  "headers": {
    "Accept": "*/*", 
    "Content-Length": "0", 
    "Host": "httpbin.default", 
    "User-Agent": "curl/7.71.1-DEV", 
    "X-B3-Sampled": "1", 
    "X-B3-Spanid": "8ad81217034f129c", 
    "X-B3-Traceid": "49473e3397096c2a8ad81217034f129c"
  }
}
``` -->