# Jai's learning-in-public lab notes

This repo is where I write up hands-on labs as I actually learn Kubernetes-adjacent tools, instead of just reading about them.

## Posts

1. [I finally understood Istio, by breaking things on purpose](./1-istio-ambient-lab-Part-1.md), a walkthrough of setting up Istio in Ambient mode, watching traffic get encrypted automatically, and then blocking traffic between two pods using an `AuthorizationPolicy`, with real commands and real output.
2. [Making one Istio installation work for many tenant clusters](./2-vcluster-istio-integration-lab-Part-2.md), setting up vCluster's Istio integration so a tenant cluster can share one Istio installation on the host, instead of installing its own.
3. [Sending 95% of traffic to one version and 5% to another, with an instant rollback](./3-vcluster-istio-canary-release-lab-Part-3.md), a canary release built entirely from inside an isolated tenant cluster, using a waypoint proxy for the actual traffic split.
4. [Letting real traffic in from outside the cluster](./4-vcluster-istio-north-south-gateway-lab-Part-4.md), exposing a tenant's service to real, external traffic through a shared ingress gateway.
5. [Making a namespace reject anything that is not properly encrypted](./5-vcluster-istio-peer-authentication-lab-Part-5.md), using `PeerAuthentication` to require encrypted traffic, and proving it with a pod deliberately kept outside the mesh.

More coming as I keep working through this.
