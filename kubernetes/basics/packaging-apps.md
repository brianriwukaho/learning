An application running on a Kubernetes cluster needs to

1. Be packaged as a container
2. Be wrapped in a pod
3. Be deployed via a declarative manifest file

# Deployments

Deployments are defined in YAML manfiest files which declaratively specify the state of the cluster.

This YAML file is posted to the API server for Kubernetes to implement