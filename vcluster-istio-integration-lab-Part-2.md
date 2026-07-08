# Making one Istio installation work for many tenant clusters

This is the follow-up to my last post, where I set up Istio in Ambient mode from scratch and proved it could encrypt traffic and block requests between two plain pods, all without touching any application code.

That post ended with a question. I work with multi-tenant Kubernetes platforms built on vCluster, where many isolated tenant clusters run on top of one shared cluster. Does every tenant need to install their own Istio, or can they all share one?

The answer is no, they can share one. This post walks through proving that, end to end, on my own laptop.

## The goal

Create a tenant cluster using vCluster. Turn on its Istio integration. Create a pod inside that tenant cluster. Watch it get synced down to the shared host cluster and automatically picked up by the same Istio installation from my last post, with zero extra setup on the tenant side.

## A quick note on licensing

The Istio integration is a vCluster Enterprise feature. It requires an active Enterprise license, applied through a running vCluster Platform instance, rather than something you can enable in a standalone, open source vCluster.

## Installing the vCluster CLI

```bash
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-darwin-arm64"
sudo install -c -m 0755 vcluster /usr/local/bin
vcluster version
```

```
vcluster version 0.33.1
```

## Installing vCluster Platform locally

Since the Istio integration needs to be licensed through the Platform, I installed vCluster Platform on my local Kind cluster, using an internal license file, `platform.yaml`, containing a license token and a list of enabled features, including `istio-integration`.

```bash
vcluster platform start --values platform.yaml
```

A couple of minutes later, the Platform UI was live at a local address, logged in as an admin automatically, running version 4.10.5. This is the same product I support customers on at work, just running on my own laptop.

## Enabling the Istio integration

Creating the tenant cluster has to go through the Platform, since that is what applies the license. I created a new tenant cluster through the Platform UI, with the following config:

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

The `integrations.istio` block is the important part. It tells vCluster's syncer to watch for `DestinationRule`, `Gateway`, and `VirtualService` objects created inside the tenant cluster, and copy them down to the Control Plane Cluster automatically.

One prerequisite worth knowing about ahead of time: the integration depends on a Kubernetes CRD called `ReferenceGrant`, part of the Gateway API, a separate CRD set from core Kubernetes and from Istio itself. It needs to be installed on the host cluster before creating the tenant cluster:

```bash
kubectl apply -f "https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml"
```

## It came up clean

```bash
kubectl get pods -A
```

```
loft-default-v-istio-tc   istio-tc-0   1/1   Running   0   54s
```

## Proving the sync actually works

I connected into the new tenant cluster:

```bash
vcluster connect istio-tc -n loft-default-v-istio-tc
```

Checked that it really is its own isolated cluster:

```bash
kubectl get namespaces
```

```
NAME              STATUS   AGE
default           Active   3m3s
kube-node-lease   Active   3m3s
kube-public       Active   3m3s
kube-system       Active   3m3s
```

No trace of anything from my host cluster or my earlier lab, as expected.

Confirmed the Istio CRDs had been installed inside the tenant automatically:

```bash
kubectl get crd | grep istio
```

```
destinationrules.networking.istio.io
gateways.networking.istio.io
virtualservices.networking.istio.io
```

Then, from inside this tenant cluster, I did the exact same thing I did in my last post: created a namespace, labeled it, deployed `httpbin`.

```bash
kubectl create namespace demo
kubectl label namespace demo istio.io/dataplane-mode=ambient
kubectl apply -f httpbin.yaml -n demo
```

One thing worth mentioning: my host cluster already had a namespace called `demo` from the earlier lab. I was worried about a naming collision, but there wasn't one. Everything created inside a tenant cluster gets synced into one single namespace already reserved for that tenant on the host, in my case `loft-default-v-istio-tc`. My tenant's `demo` namespace never becomes a real namespace on the host at all, it just becomes part of the synced object's name.

Disconnected, and looked for the pod on the host:

```bash
vcluster disconnect
kubectl get pods -n loft-default-v-istio-tc
```

```
httpbin-59b698b455-w994h-x-demo-x-istio-tc   1/1   Running   0   36s
```

That name is the translation happening in front of me. My pod's name, plus the namespace I created inside the tenant, plus the tenant cluster's own name, stitched together automatically.

Then I checked whether it actually picked up the ambient label, without me ever setting it directly on the host:

```bash
kubectl get pod -n loft-default-v-istio-tc -l app=httpbin -o jsonpath='{.items[0].metadata.labels}'
```

```json
{"app":"httpbin","istio.io/dataplane-mode":"ambient","vcluster.loft.sh/managed-by":"istio-tc","vcluster.loft.sh/namespace":"demo", ...}
```

`istio.io/dataplane-mode: ambient` is right there. I only ever set that label once, inside the tenant cluster, on a namespace called `demo`. vCluster read it and stamped the equivalent label onto the real, synced pod on the host, entirely on its own.

## The actual proof

```bash
istioctl ztunnel-config workloads | grep httpbin
```

```
demo                      httpbin-7f7cc96bdb-4f28j                     HBONE
loft-default-v-istio-tc   httpbin-...-x-demo-x-istio-tc                HBONE
```

Both `HBONE`. Both intercepted by the exact same `ztunnel`, the exact same `istiod`, the one Istio installation from my last post. One pod was created directly on the host cluster. The other came from inside a fully isolated tenant cluster that has no idea Istio even exists. Both ended up equally protected, with zero extra Istio installation and one namespace label on the tenant side.

That is the actual pitch behind this integration, proven end to end on my own laptop.
