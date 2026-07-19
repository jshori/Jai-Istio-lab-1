# I finally understood Istio, by breaking things on purpose

For a long time, Istio felt like one of those tools everyone talks about but nobody explains simply. I work in Kubernetes support every day, so I decided to stop reading about it and just build something small, on my own laptop, until it clicked.

This post is what I learned, in the order I learned it.

## What Istio actually is

Forget the marketing language for a second. Istio sits between your services and watches the traffic that flows between them. It does this without touching your application code at all.

The newer way Istio does this is called **Ambient mode**. The old way used to add an extra helper container to every single pod, which added cost and complexity. Ambient mode removes that. Instead, one shared helper called `ztunnel` runs once per machine (node), and quietly looks after every pod on that machine. Nothing in your pod changes. No extra container. No code change.

I wanted to see this with my own eyes, not just read about it.

## Setting it up

I used Kind, which just means "a small Kubernetes cluster running on my laptop." No cloud account needed.

```bash
kind create cluster --name istio-lab
```

Then I downloaded Istio's command line tool:

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.30.2
export PATH="$PWD/bin:$PATH"
```

And installed Istio with two things turned on:
- Ambient mode (the lightweight, no-sidecar approach)
- DNS capture turned off (a setting that avoids a conflict with certain platform DNS setups)

```bash
istioctl install --set profile=ambient --set values.cni.ambient.dnsCapture=false -y
```

Once installed, I checked the pods:

```bash
kubectl get pods -n istio-system
```

```
NAME                            READY   STATUS    RESTARTS   AGE
istio-cni-node-jqkcj            1/1     Running   0          52s
istiod-6d4f9d4ccd-556k2         1/1     Running   0          52s
ztunnel-v4sw4                   1/1     Running   0          42s
```

Three pieces, each with a job:
- `istiod` → the brain, decides the rules
- `ztunnel` → the helper that actually enforces them, one per node
- `istio-cni-node` → runs once per node and quietly sets up a detour, so that any traffic from an ambient pod gets sent through `ztunnel` first, before going anywhere else. It doesn't do much on its own. It just builds the road that forces traffic through the checkpoint.

Nothing exciting to look at yet. That's normal. Istio does nothing until you tell it to.

## Creating two plain pods

First, a namespace, marked so Istio knows to pay attention to anything inside it:

```bash
kubectl create namespace demo
kubectl label namespace demo istio.io/dataplane-mode=ambient
```

That one label is the entire opt-in. No sidecar injection annotations, no changes to the pods themselves.

Here's `httpbin`, a small test server that just echoes back whatever request it receives. I saved this as a file called `httpbin.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
        - name: httpbin
          image: docker.io/kong/httpbin
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
spec:
  selector:
    app: httpbin
  ports:
    - port: 80
      targetPort: 80
```

And a bare pod I can run curl commands from, saved as `curl-client.yaml`:

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

Notice neither file mentions a namespace anywhere. That's on purpose. I decide where they land at the moment I deploy them, using the `-n` flag:

```bash
kubectl apply -f httpbin.yaml -n demo
kubectl apply -f curl-client.yaml -n demo
```

The `-n demo` is what actually places them inside the `demo` namespace I created and labeled earlier. Without it, both would have quietly landed in the `default` namespace instead, and neither would have been part of the mesh, since only `demo` has the ambient label.

I checked they were up:

```bash
kubectl get pods -n demo
```

```
NAME                       READY   STATUS    RESTARTS   AGE
curl-client                1/1     Running   0          17s
httpbin-7f7cc96bdb-4f28j   1/1     Running   0          36s
```

Worth noticing: `READY` shows `1/1`, not `2/2`. In the older version of Istio, that second container would be the injected sidecar. Here, there isn't one. That's the whole point of Ambient mode: nothing about the pod changes.

## The first real "aha" moment

I ran one command, from inside `curl-client`, straight to `httpbin`:

```bash
kubectl exec -n demo curl-client -- curl -s http://httpbin/get
```

```
{"args":{},"headers":{"Accept":"*/*","Host":"httpbin","User-Agent":"curl/8.21.0"},"origin":"10.244.0.9","url":"http://httpbin/get"}
```

It worked, and returned a normal response. On the surface, this looks like nothing happened.

But it did. I checked what Istio thought was going on:

```bash
istioctl ztunnel-config workloads
```

```
NAMESPACE      POD NAME                    ADDRESS      NODE                     PROTOCOL
demo           curl-client                 10.244.0.9   istio-lab-control-plane  HBONE
demo           httpbin-7f7cc96bdb-4f28j     10.244.0.8   istio-lab-control-plane  HBONE
istio-system   ztunnel-v4sw4                10.244.0.7   istio-lab-control-plane  TCP
kube-system    coredns-7d764666f9-gkqx4     10.244.0.3   istio-lab-control-plane  TCP
```

Both pods showed up with a protocol called `HBONE`, while everything outside the mesh showed plain `TCP`. That one word, `HBONE`, was proof that my traffic had quietly been wrapped in encryption on its way between the two pods. No code change. No certificates I had to manage myself. It just happened.

This is the boring but important part of Istio: it encrypts traffic between your services automatically, without anyone having to remember to do it.

```bash
HBONE = HTTP-Based Overlay Network Encapsulation.
It's the protocol Istio's Ambient mode uses to tunnel traffic between ztunnel instances, wrapping the original connection inside an HTTP/2 CONNECT tunnel with mTLS applied.
```

## The moment it actually made sense

Encryption is nice, but it didn't feel like a "why would a company use this" moment until I tried the next thing.

I wrote a small `AuthorizationPolicy`, twelve lines of plain text, telling Istio: "curl-client is not allowed to talk to httpbin, ever."

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-curl-client
  namespace: demo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: DENY
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/demo/sa/default"]
```

I did not touch either pod. I did not add a firewall rule. I did not restart anything. I just applied this file.

I ran the exact same curl command as before:

```bash
kubectl exec -n demo curl-client -- curl -s -v http://httpbin/get
```

This time, it failed:

```
* Established connection to httpbin (10.96.200.176 port 80) from 10.244.0.9 port 45830
> GET /get HTTP/1.1
> Host: httpbin
* Request completely sent off
* Recv failure: Connection reset by peer
```

The connection was allowed to start, and the request even got sent in full. Then Istio reset it, the moment it checked who was asking.

Worth slowing down on what actually happened there, in order:
1. `ztunnel` let the TCP handshake through, no problem
2. The request went out fully, `curl` even shows `Request completely sent off`
3. `ztunnel` looked at the identity attached to that connection
4. It checked that identity against the `AuthorizationPolicy`, matched it against the `DENY` rule
5. It killed the connection right there

Notice that `curl` never got an actual HTTP response, not even a `403 Forbidden`. A `403` would mean the request reached `httpbin` and `httpbin` said no. This is a hard reset at the connection level, meaning `httpbin` never saw the request at all. httpbin's own code had no idea any of this happened. It was never given the chance to say yes or no. Istio decided that, entirely on its own, based on identity rather than an IP address or a token.

Cleaning it back up, so the two pods can talk again, is just as simple:

```bash
kubectl delete authorizationpolicy deny-curl-client -n demo
```

That's the part that made everything click for me. Companies don't adopt Istio because encryption is exciting. They adopt it because they can say things like:

- "Only the checkout service is allowed to call the payments service" (this is exactly the `AuthorizationPolicy` rule from the demo above, just applied to a real scenario)
- "All traffic between our services is encrypted, no exceptions, and no team can forget to do it"
- "We can send five percent of traffic to a new version and roll it back instantly if something looks wrong"

None of these require changing application code. They're just rules, sitting outside the application, enforced consistently everywhere.

## Where this connects to my actual job

I work with multi-tenant Kubernetes platforms built on vCluster, where many isolated tenant clusters run on top of one shared cluster. A natural question came up while I was doing this lab: does every tenant need to install their own Istio?

The answer is no, and it's a genuinely elegant design. One Istio Ambient installation runs on the shared cluster underneath everything. When someone creates a resource like a routing rule inside their own tenant cluster, it gets copied down automatically to the shared cluster, translated to the right internal names, and enforced by that one shared `ztunnel`. Every tenant gets mesh security and traffic control, without anyone installing or managing Istio themselves.

That's the part I'll be exploring next: actually installing a tenant cluster on top of this same lab, and watching a pod I create inside it get synced down and picked up by the same mesh, automatically.

## If you want to try this yourself

You don't need a cloud account or any budget. All you need is:
- Docker installed on your laptop
- Kind (a tool that runs a small Kubernetes cluster inside Docker)
- Istio's CLI tool, `istioctl`

From there, the whole lab above takes about twenty minutes, most of it just waiting for pods to start.

I'll post the follow-up once the tenant cluster part is working.
