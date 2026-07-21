# Making a namespace reject anything that is not properly encrypted

This is the fifth post in this series. The earlier posts covered automatic encryption, sharing one Istio installation across tenant clusters, splitting traffic between two versions of a service, and letting real traffic in from outside the cluster. This post is about a different kind of control: deciding whether a namespace accepts plain, unencrypted traffic at all, or only traffic that is properly encrypted.

The object that controls this is called `PeerAuthentication`. It answers a different question than the `AuthorizationPolicy` object from the first post. `AuthorizationPolicy` decides who is allowed to talk to a service, and what they are allowed to do once they are talking to it. `PeerAuthentication` decides something different: how strict the encryption requirement is for anyone talking to a namespace, regardless of who they are or what they are trying to do.

`PeerAuthentication` has a few modes, but the two that matter here are `PERMISSIVE`, which accepts both encrypted and plain traffic, and `STRICT`, which only accepts properly encrypted traffic and rejects everything else. In ambient mode, `PERMISSIVE` is actually the default already in place, since ztunnel automatically encrypts traffic between mesh pods anyway.

## Why this needed something new to actually see it working

Every pod in this series so far has lived inside a namespace labeled for the mesh, so any traffic between them was always automatically encrypted already. That means switching between `PERMISSIVE` and `STRICT` would have looked identical, since there was never any plain traffic in the picture to begin with.

To actually see the difference, I needed a pod that is not part of the mesh at all, sending genuinely plain, unencrypted traffic on purpose.

## Creating a pod outside the mesh

Connected to the tenant cluster, I created a namespace with no ambient label on it:

```bash
vcluster connect tc-istio -n loft-default-v-tc-istio
```

```bash
kubectl create namespace outside-mesh
```

I saved this as `external-client.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: external-client
spec:
  containers:
    - name: curl
      image: curlimages/curl
      command: ["sleep", "infinity"]
```

```bash
kubectl apply -f external-client.yaml -n outside-mesh
```

```bash
kubectl get pods -n outside-mesh
```

```
NAME              READY   STATUS    RESTARTS   AGE
external-client   1/1     Running   0          5s
```

Since `outside-mesh` was never labeled `istio.io/dataplane-mode=ambient`, anything sent from this pod leaves as plain, ordinary HTTP, not wrapped in the encrypted tunnel our other pods use automatically.

## Confirming plain traffic gets through by default

```bash
kubectl exec -n outside-mesh external-client -- curl -s http://nginx.demo.svc.cluster.local
```

```
<h1>Version 1 (stable)</h1>
```

A real, plain request from a pod outside the mesh reached `nginx` without any trouble. This confirms the default `PERMISSIVE` behavior, encrypted and plain traffic are both accepted.

## Where PeerAuthentication actually needs to be applied

This one needs to be applied directly on the host cluster, not inside the tenant. vCluster's Istio integration only pre-installs three CRDs into a tenant, `DestinationRule`, `Gateway`, and `VirtualService`, `PeerAuthentication` is not one of them. The host cluster's full Istio install already knows this object, and the actual enforcement happens there anyway.

I saved this as `peer-authentication-strict.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: loft-default-v-tc-istio
spec:
  mtls:
    mode: STRICT
```

Two things worth pointing out in this file. First, `name: default`, with no `selector` field. This is a specific Istio convention: a `PeerAuthentication` named exactly `default` applies to every pod in that namespace, not just one. Second, `namespace: loft-default-v-tc-istio`. This is not the tenant's own name for its namespace, it is the real, synced namespace on the host, the one where `nginx`'s actual pods physically live.

```bash
kubectl config use-context kind-istio-lab
kubectl apply -f peer-authentication-strict.yaml
```

## Testing both sides

First, the pod outside the mesh, the exact same request as before:

```bash
kubectl config use-context vcluster_tc-istio_loft-default-v-tc-istio_kind-istio-lab
kubectl exec -n outside-mesh external-client -- curl -s http://nginx.demo.svc.cluster.local -v
```

```
* Established connection to nginx.demo.svc.cluster.local (10.96.17.19 port 80) from 10.244.0.58 port 40290
> GET / HTTP/1.1
> Host: nginx.demo.svc.cluster.local
* Request completely sent off
* Recv failure: Connection reset by peer
command terminated with exit code 56
```

The connection opened, the request went out completely, and then got reset. This is the same pattern from the very first post's `AuthorizationPolicy` demo, the request never actually reached `nginx` at all, `ztunnel` rejected it purely because it was not properly encrypted mesh traffic.

Then, a pod that genuinely is part of the mesh:

```bash
kubectl exec -n demo curl-client -- curl -s http://nginx.demo.svc.cluster.local -v
```

```
* Established connection to nginx.demo.svc.cluster.local (10.96.17.19 port 80) from 10.244.0.51 port 37144
> GET / HTTP/1.1
< HTTP/1.1 200 OK
< server: istio-envoy
<h1>Version 1 (stable)</h1>
* Connection #0 to host nginx.demo.svc.cluster.local:80 left intact
```

A normal, successful response, `server: istio-envoy` in the headers confirms this traffic properly passed through the mesh.

Same policy, same namespace, same destination service. One caller, properly part of the mesh, got through without any trouble. The other, deliberately outside the mesh, was rejected before its request ever reached the application. That is the entire point of `PeerAuthentication` in `STRICT` mode, shown with two real, opposite outcomes from the same rule.

One more small thing worth knowing. This `PeerAuthentication` only applied to one namespace, since that is where it was created. This is the normal, default behavior. If you want a rule like this to apply everywhere in the mesh, to every namespace at once, you place it in a special namespace instead, usually called `istio-system`. That one namespace is treated as the mesh-wide spot, so anything created there applies to all namespaces, not just one.
