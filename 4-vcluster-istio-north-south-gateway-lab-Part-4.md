# Letting real traffic in from outside the cluster

This is the fourth post in this series. So far, every test I have run has been traffic that starts and ends inside the cluster, one pod calling another pod. This post is about the opposite case: a real request starting from outside the cluster, like a request from a browser or a curl command run on my own laptop, reaching a service running inside an isolated tenant cluster.

This kind of traffic, coming in from outside the cluster, has a name: north-south traffic. Everything we built in the earlier posts, one service calling another service inside the same cluster, is called east-west traffic. This post is specifically about the north-south kind.

## Two different things both called Gateway

Before writing anything, it is worth being precise about a word that comes up a lot here: Gateway. There are two separate things with this name, and mixing them up caused me real confusion while building this.

Istio has its own Gateway, from `networking.istio.io`. This one is just a written rule, similar to a VirtualService or DestinationRule. It does not create anything on its own. It only works by pointing at a gateway deployment that already exists.

The Kubernetes Gateway API also has a Gateway, from `gateway.networking.k8s.io`. This is a different, newer standard, and creating one of these can automatically build a brand new deployment for you, the same thing that happened when we set up a waypoint in the last post.

This post uses the first kind, Istio's own Gateway, since the goal here was to point at one already-existing, shared entry point, not create a new one.

## Installing a real entry point

Before this post, our cluster had no real front door at all. We had installed Istio using the ambient profile, which sets up the internal parts of the mesh, but does not include anything that listens for traffic coming in from outside.

The actual program that does this job is Envoy, the same proxy software used everywhere else in Istio, just running here in a different role. When Envoy is deployed specifically to accept traffic from outside the cluster, it is called an ingress gateway.

```bash
kubectl config use-context kind-istio-lab
istioctl install --set profile=ambient --set 'components.ingressGateways[0].name=istio-ingressgateway' --set 'components.ingressGateways[0].enabled=true' -y
```

`--set profile=ambient` keeps everything already installed exactly as it is. The two new lines are what actually add something: they tell Istio to also build one ingress gateway deployment, and name it `istio-ingressgateway`. That name is not special, it is just the name Istio's own examples commonly use.

```bash
kubectl get pods -n istio-system
```

```
NAME                                   READY   STATUS    RESTARTS   AGE
istio-cni-node-jqkcj                   1/1     Running   0          13d
istio-ingressgateway-9c6cc9fdf-r289v   1/1     Running   0          2m58s
istiod-6d4f9d4ccd-556k2                1/1     Running   0          13d
ztunnel-v4sw4                          1/1     Running   0          13d
```

A new pod appears, everything else stays untouched. This ingress gateway is a shared, one-time piece of infrastructure, installed once by the platform team on the host, the same category as `istiod` and `ztunnel`, not something created per tenant.

## Turning on Gateway syncing for this tenant cluster

Since Istio's own Gateway is just a written rule, a tenant needs a way to write one and have it relayed to the host, the same way DestinationRule and VirtualService already work. This requires one change to the tenant's `vcluster.yaml`:

```yaml
sync:
  toHost:
    ingresses:
      enabled: true
controlPlane:
  coredns:
    enabled: true
    embedded: true
  backingStore:
    etcd:
      embedded:
        enabled: true
integrations:
  istio:
    enabled: true
    sync:
      toHost:
        destinationRules:
          enabled: true
        gateways:
          enabled: true
        virtualServices:
          enabled: true
```

The only change from the last post is `gateways.enabled`, now set to `true`. Applying this redeploys the tenant's own control-plane pod for a few seconds, but the tenant's actual application pods keep running the whole time.

```bash
kubectl get pods -n loft-default-v-tc-istio
```

```
NAME                                          READY   STATUS    RESTARTS   AGE
curl-client-x-demo-x-tc-istio                 1/1     Running   0          47h
nginx-v1-54bdc8b467-65sx2-x-demo-x-tc-istio   1/1     Running   0          47h
nginx-v2-fb4f84b89-mp8p9-x-demo-x-tc-istio    1/1     Running   0          47h
tc-istio-0                                    1/1     Running   0          2m
waypoint-b69f5756b-cbvqm                      1/1     Running   0          47h
```

Only `tc-istio-0`, the control plane, restarted. Everything else was undisturbed.

## Writing the Gateway, from inside the tenant

Connected to the tenant cluster:

```bash
vcluster connect tc-istio -n loft-default-v-tc-istio
```

I saved this as `tc-istio-gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: tc-istio-gateway
  namespace: demo
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

```bash
kubectl apply -f tc-istio-gateway.yaml
```

`selector: istio: ingressgateway` is the actual connection this whole file makes. It says "attach this rule to whichever real deployment carries the label `istio: ingressgateway`", which is exactly the label on the shared ingress gateway we installed earlier. This file creates nothing new by itself, it only points at something that already exists.

## Writing the VirtualService that actually routes the traffic

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: nginx-external
  namespace: demo
spec:
  hosts:
    - "*"
  gateways:
    - tc-istio-gateway
  http:
    - route:
        - destination:
            host: nginx.demo.svc.cluster.local
            port:
              number: 80
```

```bash
kubectl apply -f nginx-external-virtualservice.yaml
```

The one new piece here, compared to the VirtualServices from the earlier posts, is `gateways: [tc-istio-gateway]`. Without this line, a VirtualService only ever applies to traffic already inside the mesh. This line is what tells it "also apply this rule to traffic entering through the gateway I just made."

Checked on the host that it synced correctly:

```bash
kubectl config use-context kind-istio-lab
kubectl get gateway.networking.istio.io -n loft-default-v-tc-istio
```

```
NAME             AGE
v1ep8skh4o51qg   77s
```

## Testing it from outside the mesh

Since this Kind cluster has no real cloud load balancer to hand out a public IP, I used port-forward to reach the ingress gateway from my own laptop, standing in for a real external client:

```bash
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
```

Left that running, and in a second terminal:

```bash
curl -s http://localhost:8080/
```

```
<h1>Version 2 (canary)</h1>
```

Here is exactly what happened to that one request, step by step.

1. The `curl` command ran on my own laptop, completely outside the cluster.
2. `port-forward` picked it up and handed it to the `istio-ingressgateway` pod on the host, since that pod is the only thing actually listening on port 80 that my laptop can reach through the forwarded connection.
3. The gateway checked its own rules and found `tc-istio-gateway`, since that Gateway object told it "accept traffic on port 80 for any hostname."
4. It then checked which VirtualService is attached to that Gateway, and found `nginx-external`, since that VirtualService listed `tc-istio-gateway` under its own `gateways` field.
5. That VirtualService's rule pointed at `nginx.demo.svc.cluster.local`, the tenant's own name for the service. On the host, this had already been translated to the real, synced service name, `nginx-x-demo-x-tc-istio`.
6. The gateway then opened a connection straight to one of the real `nginx` pods, `nginx-v1` or `nginx-v2`, through `ztunnel`, the same component from the first post in this series, which encrypted the connection and checked both sides' identity before letting the traffic through.
7. The `nginx` pod answered, and that response, one version or the other, traveled all the way back the same path, arriving in my terminal.

Running the request a few more times showed a mix of both versions, confirming the same 95 percent to 5 percent canary split from the earlier post was still active, now deciding traffic that started completely outside the cluster.

## What's next

The next post in this series moves into security policy: specifically, `PeerAuthentication`, the object that decides whether a namespace accepts only encrypted traffic, or allows plain, unencrypted traffic alongside it. The plan is to first see plain traffic get through in a relaxed mode, then flip that same namespace to a strict mode and watch that same plain traffic get rejected, all without touching any application code.
