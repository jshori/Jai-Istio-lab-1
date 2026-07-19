# Sending 95% of traffic to one version and 5% to another, with an instant rollback

This is the third post in this series. In the first, I set up Istio in Ambient mode from scratch and proved it encrypts traffic and enforces access rules automatically. In the second, I proved a vCluster tenant cluster can share that same Istio installation, without installing its own copy.

This post tackles a real use case I mentioned in the first post.

Here is the idea, with an everyday example. Imagine a company has an internal checkout service, and that checkout service needs to call a payment service to process an order. The team building the payment service wants to release a new version, but instead of switching every single call over at once, which is risky if something is broken, they do something safer. They let a small number of calls, say 5 out of every 100, quietly go to the new version, while the rest keep going to the version that already works fine. If the new version turns out to have a bug, the team can switch every call straight back to the old version in seconds, no code changes, no restarting anything, just flipping a setting. This is a common, everyday practice at real companies, and it has a name: a canary release.

This kind of traffic, one internal service calling another internal service, all inside the same cluster, is sometimes called east-west traffic, as opposed to traffic coming in from outside the cluster, like a real customer's browser, which is called north-south traffic. This post is specifically about the east-west kind, splitting traffic between two internal services.

## The two building blocks: VirtualService and DestinationRule

Before touching any commands, it helps to know what these two objects actually do, since the whole demo is just these two working together.

**VirtualService** decides where traffic should go. It is the actual routing rule, the thing that says "send this percentage here, that percentage there."

**DestinationRule** works alongside a VirtualService. It groups pods into smaller, named groups called subsets, based on their labels, and this is how it defines what a "version" of a service actually means. For example, it can say "any pod labeled `version: v1` belongs to a group called v1." A VirtualService can only send traffic to a subset if a DestinationRule has already defined that subset, otherwise the name would not mean anything.

Both of these are just written rules and sit there as instructions.

## Creating a fresh tenant cluster for this lab

Before touching the app itself, I created a new tenant cluster through vCluster Platform, called `tc-istio`, with the same Istio integration settings from the last post:

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
          enabled: false
        virtualServices:
          enabled: true
```

This is the exact same config style from Part 2. One thing worth calling out: `gateways` under the Istio integration is set to `false`. This is Istio's own native Gateway object, used for letting traffic in from the public internet, which is a completely different thing from the Kubernetes Gateway API object the waypoint uses later in this post. Since this lab never exposes anything to the outside internet, there is nothing for it to do, so it stays off.

Creating it through the Platform, rather than the plain vCluster CLI, is what applies the Enterprise license the Istio integration needs. Once it showed `Running`, I connected into it:

```bash
vcluster connect tc-istio -n loft-default-v-tc-istio
```

## Setting up two versions of the same app

To keep the lab simple, I did not use a real checkout or payment service. I used two versions of plain nginx instead, each serving different content so I could tell which one answered a request. The idea is the same either way, one service, two versions, calling it from another pod inside the same cluster.

First, a namespace, labeled for the mesh:

```bash
kubectl create namespace demo
kubectl label namespace demo istio.io/dataplane-mode=ambient
```

A ConfigMap per version, so each serves different content:

```bash
kubectl create configmap nginx-v1 -n demo --from-literal=index.html="<h1>Version 1 (stable)</h1>"
kubectl create configmap nginx-v2 -n demo --from-literal=index.html="<h1>Version 2 (canary)</h1>"
```

Two Deployments, one Service. I saved this as `nginx-canary.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
      volumes:
        - name: html
          configMap:
            name: nginx-v1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: v2
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
      volumes:
        - name: html
          configMap:
            name: nginx-v2
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
```

Then deployed it:

```bash
kubectl apply -f nginx-canary.yaml -n demo
```

Notice both Deployments carry the label `app: nginx`, but differ on `version: v1` / `version: v2`. The Service selects only on `app: nginx`, so it treats both as valid backends. This is exactly what lets a DestinationRule later tell them apart.

I also created a plain curl pod to test with. I saved this as `curl-client.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-client
spec:
  containers:
    - name: curl
      image: curlimages/curl
      command: ["sleep", "infinity"]
```

```bash
kubectl apply -f curl-client.yaml -n demo
```

```bash
kubectl get pods -n demo
```

```
NAME                        READY   STATUS    RESTARTS   AGE
curl-client                 1/1     Running   0          34s
nginx-v1-54bdc8b467-65sx2   1/1     Running   0          19s
nginx-v2-fb4f84b89-mp8p9    1/1     Running   0          19s
```

## The DestinationRule: naming the two versions

First, a quick word on what a "subset" actually means, since the DestinationRule below uses that word a lot. A subset is just a name you give to a smaller group of pods within a bigger service, based on a label. Right now, the `nginx` service points at two different groups of pods, the `v1` pods and the `v2` pods. Kubernetes itself has no idea these two groups are meant to be treated differently, to Kubernetes, they are all just "nginx" pods. A subset is how you tell Istio "actually, treat these two groups separately, and call them v1 and v2 so I can refer to them by name later."

I saved this as `nginx-destinationrule.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: nginx-destination
spec:
  host: nginx.demo.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

```bash
kubectl apply -f nginx-destinationrule.yaml -n demo
```

Let's go through this line by line.

`host: nginx.demo.svc.cluster.local` just tells Istio which Service this rule is about. It is the same as writing `nginx.demo.svc.cluster.local` in a browser, just the internal address for our `nginx` Service.

The `subsets` part is where the real work happens. Right now, if you just look at your pods, Kubernetes has no idea that `nginx-v1` and `nginx-v2` are supposed to be treated differently. To Kubernetes, they are both just "nginx" pods.

So we tell Istio ourselves, in plain words: "any pod that has the label `version: v1` on it, put it in a group, and call that group `v1`." That is exactly what these four lines say:

```yaml
    - name: v1
      labels:
        version: v1
```

We do the same thing for `v2` right below it.

Before this file existed, the label `version: v1` was just some text sitting on a pod. It did not mean anything special. After this file is applied, Istio now knows that "v1" is a real group of pods it can send traffic to. That is the only job this file does, nothing more.

## The VirtualService: the actual 95/5 split

I saved this as `nginx-virtualservice.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: nginx
spec:
  hosts:
    - nginx.demo.svc.cluster.local
  http:
    - route:
        - destination:
            host: nginx.demo.svc.cluster.local
            subset: v1
          weight: 95
        - destination:
            host: nginx.demo.svc.cluster.local
            subset: v2
          weight: 5
```

```bash
kubectl apply -f nginx-virtualservice.yaml -n demo
```

`weight: 95` and `weight: 5` are the entire canary release, right here. Istio randomly routes each incoming request according to these percentages, per request, with no application code involved.

Both of these objects were applied from inside the tenant cluster, exactly like we did with the earlier Istio labs. They synced down automatically, same mechanism as before.

## The missing piece: a waypoint proxy

Weighted routing like this is an L7 feature. It has to actually read the HTTP request to decide where it goes. In Ambient mode, the always-on `ztunnel` component only handles L4, encryption and identity, and deliberately never reads HTTP. So a VirtualService by itself does nothing until something capable of L7 decisions is in the traffic path. That something is called a waypoint proxy, a real Envoy proxy that Istio deploys specifically to make these decisions.

A waypoint is scoped to a namespace. After a namespace is set up to use one, requests to any service in that namespace get routed through it for L7 processing. This is an important distinction from VirtualService and DestinationRule: those two are pure configuration with nothing running behind them. A waypoint is real, running infrastructure, an actual pod consuming real CPU and memory.

That distinction is also why it belongs in a different place than the VirtualService and DestinationRule.

## Why the waypoint goes on the host, not inside the tenant

Everything we have proven since the first post rests on one rule: real running infrastructure, like `istiod` and `ztunnel`, only ever runs once, on the shared host. Tenants only ever author configuration, which gets relayed to that shared infrastructure.

A waypoint is infrastructure, not configuration, so it follows the same rule. It gets created directly on the host, targeting the exact namespace where the tenant's actual pods physically live once synced, in this case `loft-default-v-tc-istio`.

```bash
kubectl config use-context kind-istio-lab
istioctl waypoint apply -n loft-default-v-tc-istio --enroll-namespace
```

Breaking down the command:
- `istioctl waypoint apply` creates the waypoint. Behind the scenes, this is a Gateway object, from the Kubernetes Gateway API, a different thing entirely from Istio's own native Gateway object used for public internet ingress. Istio sees this specific kind of Gateway and provisions a real Envoy pod for it.
- `-n loft-default-v-tc-istio` says which namespace this waypoint handles. Since that is where the tenant's synced pods actually live on the host, this is where it has to go.
- `--enroll-namespace` also labels that namespace to say "route traffic through this waypoint," so no separate labeling step is needed.

```bash
kubectl get gateway.gateway.networking.k8s.io -n loft-default-v-tc-istio
```

```
NAME       CLASS            ADDRESS        PROGRAMMED   AGE
waypoint   istio-waypoint   10.96.81.171   True         7s
```

`PROGRAMMED: True` confirms it is actually working, not just created.

```bash
kubectl get pods -n loft-default-v-tc-istio
```

```
NAME                                          READY   STATUS    RESTARTS   AGE
curl-client-x-demo-x-tc-istio                 1/1     Running   0          4m44s
nginx-v1-54bdc8b467-65sx2-x-demo-x-tc-istio   1/1     Running   0          4m29s
nginx-v2-fb4f84b89-mp8p9-x-demo-x-tc-istio    1/1     Running   0          4m29s
tc-istio-0                                    1/1     Running   0          9m45s
waypoint-b69f5756b-cbvqm                      1/1     Running   0          13s
```

A single, stable waypoint pod, sitting right alongside the tenant's synced workloads.

## Running the actual test

Back in the tenant cluster:

```bash
kubectl exec -n demo curl-client -- sh -c 'for i in $(seq 1 500); do curl -s nginx; echo; done' | sort | uniq -c
```

```
 475 <h1>Version 1 (stable)</h1>
  25 <h1>Version 2 (canary)</h1>
```

475 out of 500 landed on the stable version, 25 on the canary, an exact 95/5 split. A smaller batch of 100 requests will not always land on exactly 95/5, since each request is an independent random choice, but the ratio converges closer to 95/5 the more requests you send.

## The instant rollback

If the canary version showed a problem in production, here is the entire fix, one config change, no redeploy, no pod restart:

I saved this as `nginx-virtualservice.yaml`, overwriting the earlier version:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: nginx
spec:
  hosts:
    - nginx.demo.svc.cluster.local
  http:
    - route:
        - destination:
            host: nginx.demo.svc.cluster.local
            subset: v1
          weight: 100
        - destination:
            host: nginx.demo.svc.cluster.local
            subset: v2
          weight: 0
```

```bash
kubectl apply -f nginx-virtualservice.yaml -n demo
```

Only the two weight values changed. Applying it updates the existing rule in place.

```bash
kubectl exec -n demo curl-client -- sh -c 'for i in $(seq 1 20); do curl -s nginx; echo " -- request $i"; sleep 0.3; done'
```

Every single one of 20 requests came back as Version 1. Zero traffic reached the canary version, immediately, with nothing else touched.

## Why this matters

This is the real value behind "try it on a few people first, and undo it instantly if needed." A developer inside an isolated tenant cluster wrote two plain Kubernetes objects, exactly like they would in any normal cluster with Istio. A waypoint, set up once by the platform team on the shared host, did the actual work of splitting the traffic. And undoing it was a single small change, not a new deployment, not downtime, not waiting on anything to restart.
