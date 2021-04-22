## Services Theory

Individual Pod IPs cannot be relied on. To solve this issue, Services are required as they provide stable and reliable networking for a set of Dynamic Pods.

Every Service gets its own stable IP address, DNS name and port

# Labels and Loose Coupling

Services are loosely coupled with Pods via labels and label selectors. This is the same system that loosely couples Deployments to Pods.

The following Service and Deployment YAMLs show how selectors and labels are implemented

```
apiVersion: v1
kind: Service
metadata:
  name: test-svc
spec:
  ports:
  - port: 8080
  selector:
    app: hello-world   # Label selector
    # Service is looking for Pods with the label `app=hello-world`
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world   # Pod labels
        # The label matches the Service's label selector
    spec:
      containers:
      - name: hello-ctr
        image: test/test:latest
        ports:
        - containerPort: 8080
```

In these example files, the service has a selector with a single value `app=hello-world`. This is the label the services looks for when querying the cluster for matching Pods. 

The Deployment specifies a Pod template with the same `app=hello-world` label. As a result, any Pods it deploys will have the `app=hello-world` label.

When the Deployment and Service are deployed, the service will select all 10 Pod replicas and provide a stable networking endpoint and load balance traffic to them.

## Services and Endpoints Objects

As Pods are created and deleted, the Service dynamically updates its list of healthy matching Pods. This is done through a combination of label selectors and a construct called the Endpoints object.

Each Service that is created automatically gets an associated Endpoints object. All this Object is, is a dynamic list of all of the healthy Pods on the cluster that match the Service's label Selector.

When traffic is sent to Pods via a Service, an app will normally query the cluster's internal DNS for the IP address of a Service. It then sends the traffic to this IP address which will have the Service send it to a Pod.

A Kubernetes-native application (an app that understands K8s and can use the K8s API) can query the Endpoints API directly and bypass DNS lookup.

## Accessing Services from inside the Cluster

Kubernetes supports several type of Services. The default type is **ClusterIP**

A ClusterIP Service has a stable IP address and port that is only accessible from inside the cluster.  The ClusterIP gets registered against the name of the Service on the cluster's internal DNS service. 

All Pods in a cluster are preprogrammed to know about the cluster's DNS service which means all Pods are able to resolve Service names.

As long as a Pod knows the name of a Service, it can resolve that to its ClusterIP address and connect to the desired Pods.

## Accessing Services from outside the Cluster

Kubernetes has another type of Service called `NodePort`. This type builds on top of ClusterIP and enables access from outside the cluster.

A NodePort builds on the ClusterIP by adding another port that can be used to reach Service from outside the cluster. This additional port is called the NodePort

## Other types of Services.

The other Service Types are `LoadBalancer` and `ExternalName`.

LoadBalancer Services integrate with load balancers from cloud providers. They build on top of NodePort Services and allow clients on the internet to reach Pods via cloud load balancers. 

ExternalName services route traffic to systems outside of your Kubernetes cluster.

## Service Discovery

Kubernetes Service discovery can be implemented via DNS (preffered) or Environment Variables (don't do this)