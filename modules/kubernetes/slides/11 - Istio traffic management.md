# 11 - Istio traffic management

Istio’s traffic management relies on Envoy proxies deployed as sidecars alongside your services. These proxies intercept 
and mediate all inbound and outbound data plane traffic, allowing you to orchestrate the flow of traffic around your 
service mesh without having to make any changes to your application code.

Below is an overview of the core API resources and concepts used to configure Istio traffic management.

## 1. Gateway
A Gateway is used to manage inbound and outbound traffic for your mesh. Unlike other Envoy proxies that run as sidecars 
next to your application workloads, gateway configurations are applied to standalone Envoy proxies operating at the edge 
of the mesh.

Gateways allow you to specify exact parameters for traffic entering or leaving the mesh, configuring Layer 4-6 load 
balancing properties such as exposed ports, protocols, and TLS settings.

### YAML Declaration
Instead of adding application-layer traffic routing (Layer 7) directly to the gateway, Istio lets you bind a regular 
Virtual Service to the gateway. Here is an example of an ingress gateway configured to accept external HTTPS traffic:
```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      credentialName: ext-host-cert
```

## 2. Virtual Services
Virtual services and destination rules are the primary building blocks of Istio's traffic routing functionality. A 
Virtual Service strongly decouples the user-facing destination that clients send requests to from the actual backend 
workloads that implement the service.

### YAML Declaration
The following virtual service routes traffic to different versions of a reviews service depending on whether the request 
comes from a specific user:
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

### Hosts
The hosts field lists the user-addressable destinations the routing rules apply to. These are the addresses clients use 
when sending requests. The host can be an IP address, a DNS name, a wildcard prefix, or a short name (like a Kubernetes 
service name). They do not actually need to exist in the Istio service registry, allowing you to model abstract virtual 
destinations.

### Routing Rules
The http section contains the routing rules that dictate how traffic is evaluated and forwarded:
- **Match Condition**: Evaluates whether a request matches the rule. In the example above, the match field looks for an exact 
  end-user HTTP header value of jason. Conditions can be based on URIs, headers, ports, and more.  
- **Destination**: If a condition is met, the traffic is sent to the specified destination. Unlike the virtual host, this 
  must be a real destination in Istio's service registry. 
- **Routing Rule Precedence**: Rules are evaluated sequentially from top to bottom. The first rule to match a request takes 
  priority. It is highly recommended to add a "no condition" default rule at the bottom (like subset: v3 in the example)
  to ensure unmatched traffic has a route.

## 3. Destination Rule
If Virtual Services determine where traffic is routed, Destination Rules configure what happens to the traffic once it 
reaches that destination. They are applied after virtual service routing rules are evaluated.

### YAML Declaration
Here is an example of a Destination Rule configuring subsets and assigning different load balancing options:
```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

### Subsets
Subsets define specific groupings of your service's instances, frequently used for A/B testing or canary deployments. 
Each subset is mapped to a Kubernetes deployment using labels (key/value metadata pairs attached to your Pods). Once your 
subsets are defined in a Destination Rule, your Virtual Services can safely route traffic to them.

### Load Balancing Options
By default, Envoy proxies distribute traffic using a least requests model (routing to the host with the fewest active 
requests). Destination Rules allow you to override this with other algorithms, either globally for the service or on a 
per-subset basis:
- **Random**: Forwards requests entirely at random.
- **Weighted**: Directs a specific percentage of traffic to instances.
- **Round Robin**: Forwards requests to each instance in strict sequence.  
- **Consistent Hash**: Provides soft session affinity based on HTTP headers, cookies, or source IP using algorithms like Ring Hash or Maglev.

## Appendix: Network Resilience and Testing
Istio provides out-of-the-box mechanisms to build network resilience and test failure recovery transparently, completely 
independent of your application logic.

### Timeouts
A timeout dictates the maximum amount of time an Envoy proxy should wait for a reply from a given service. Configuring 
timeouts ensures calls succeed or fail within a predictable window rather than hanging indefinitely. You can configure 
timeouts dynamically on a per-service basis via a Virtual Service:
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
    - ratings
  http:
    - route:
        - destination:
            host: ratings
            subset: v1
      timeout: 10s
```

### Retries
Retries improve service availability by specifying the maximum number of times an Envoy proxy should attempt to connect 
to a service if the initial call fails. Istio handles the interval between retries automatically to ensure the failing 
service isn’t overwhelmed by traffic.
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
```

### Circuit Breakers
Circuit breaking is a technique to prevent localized transient failures from cascading to other nodes in the mesh. 
Circuit breakers are configured in DestinationRule resources. You can limit the impact of failures by restricting the 
number of concurrent connections or requests allowed to a service:
```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          tcp:
            maxConnections: 100
```

### Fault Injection
To test your application's failure recovery policies, Istio allows you to deliberately inject faults at the application
layer. This helps identify restrictive or incompatible recovery configurations before they cause an outage. (Note: Fault
injection currently cannot be combined with retries or timeouts on the same Virtual Service).

You can inject two types of faults using a Virtual Service:
- **Delays**: Timing failures that mimic increased network latency or overloaded upstream services.
- **Aborts**: Crash failures that mimic upstream service failures by returning HTTP error codes.

Example of injecting a 5-second delay for 0.1% of requests:
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```