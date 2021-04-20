# Pods 

In VMware, the atomic unit of scheduling is the virtual machine (VM). In Docker it is the container. In Kubernetes it is the Pod.

An application cannot run directly on a Kubernetes cluster, but must be run inside of a Pod.

## Pods and containers

Since the Docker logo is a whale, a group of containers is called a Pod which is plural for a gorup of whales.

The simplest Pod configuration is to run a single container per Pod. Some common multi-container Pod configurations are:

* Service Meshes
* Web containers supported by helper containers (?)
* Containers with a tightly coupled log scraper

The Kubernetes pod is a construct for running one or more containers

## Pod anatomy

A pod is a ring-fenced environment to run containers. The Pod does not run the containers, but is just a sandbox for hosting containers.

Multiple containers in a Pod share the same Pod Environment. This includes IPC namespace, shared memory, volumes, and network stack. Containers in the same Pod can talk to eachother via the Pod's localhost interface. 

e.g. all containers in the same pod will share the same IP address

If you do not need to tightly couple your containers, they should be put in their own Pods and be looosely coupled over the network.

## Pods as the unit of scaling

Pods are also the minimum unit of scaling in Kubernetes. Apps are scaled by adding or removing Pods, not by manipulating containers in existing Pods.

## Pods: atomic operations

The deployment of a pod is an atomic operation. A Pod is only considered ready for service when all of its containers are up and running. A partially deployed pod will never service requests.

## Pod lifecycle

When a Pod "dies", Kubernetes starts a new one to replace it with a new ID and IP address.

When designing applications do not couple them tightly to one particular Pod instance. Build with graceful degradation and data consistency in mind.