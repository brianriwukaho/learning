# Kubernetes DNS

Every Kubernetes cluster has an internal DNS service. It has a static IP address hard-coded into every Pod on the Cluster so that all containers and Pods know how to find it.

Cluster DNS is based on CoreDNS (https://cordens.io)
