# Nodes

Nodes are the workers of a Kubernetes cluster and do 3 things:

1. Watch the API server for new tasks
2. Execute tasks
3. Report back to the control pane

There are three major components to a node: Kubelet, Container Runtime, Kube-proxy

## Kubelet

The main Kubernetes agent. One of it's main jobs is to watch the API server for new tasks and maintains a reporting channel to to control plane.

If the kubelet cannot execute a task it reports back to the Control plane which will decide what to do.

## Container Runtime

Required to perform container-related tasks: pulling images, start and stopping container etc.

Masks the internals of Kubernetes and exposes an interface for third-party container runtimes. Here is an example of a popular one https://containerd.io/

## Kube-proxy

Responsible for local clutser networking. Ensures each node gets its own unique I{ address and implements local IPTABLES or IPVS rules to handle routing and load balancing
