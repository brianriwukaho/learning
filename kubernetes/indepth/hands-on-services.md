# Interacting with Services

## Imperatively creating a Kubernetes Service

**The Imperative way is not the Kubernetes way**. It introduces the risk that stale manifests are subsequently used to update the cluster, unintentionally overwriting important changes that were made imperatively.

Use `kubectl` to declaratively deploy the following Deployment.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-ctr
        image: test/test:latest
        ports:
        - containerPort: 8080
```

Deploy with the following command

```
kubectl apply -f deploy.yml
```

Which will output the following:

```
deployment.apps/web-deploy created
```

The command to imperatively create a Kubernetes Service is `kubectl expose`

```
$ kubectl expose deployment web-deploy \
  --name=hello-svc \
  --target-port=8080 \
  --type=NodePort
```

Which will output the following:

```
service/hello-svc exposed
```

Once the service is created, it can be inspected with `kubectl describe svc hello-svc`

```
kubectl describe svc hello-svc
```

Which will output the following: 

```
Name:                     hello-svc
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=hello-world
Type:                     NodePort
IP:                       192.168.201.116
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset> 30175/TCP
Endpoints:                192.168.128.13:8080,192.168.128.249:8080, + more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Notable output values include:
* `Selector`: is the list of labels that Pods must have in order for the Service to send traffic to them.
* `IP`: is the permanent ClusterIP (VIP) of the service
* `Port`: is the port that t    he Service listens on inside the cluster
* `TargetPort`: is the port that the application is listening on
* `NodePort`: is the cluster-wide port that can be used to access it from outside the cluster
* `Endpoints`: is the dynamic list of healthy Pod IPs that currently match the Service's label selector

## Declarative creating a Kubernetes Service

Below is an example manifest file that would be used to Deploy a Kubernetes service

```
apiVersion: v1
kind: Service
metadata:
  name: test-svc
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
    targetPort: 8080
    protocol: TCP
  selector:
    app: hello-world
```

* The `apiVersion` field specifies what core API group the object we are creating is located in.
* The `.kind` field tells Kubernetes that we are defining a Service object
* The `.spec` section is where the Service is defined.  
  * The `port` value configures the Service to listen on port `8080` for internal requests
  * The `NodePort` value tells the Service to listen on 30001 for external requests
  * The `targetPort` value tells Kubernetes to send traffic to application Pods on port `8080`
  * The `protocol` value specifies use of `TCP` (default)
  * The selector value tell the Service to send traffic to all Pods in the cluster that have the `app=hello-world` label.

## Common Service types

The three common ServiceTypes are:
* `ClusterIP` - This is the default ServiceType and gives the Service a stable IP address internally within the cluster. This will not make the service available outside of the cluster.
* `NodePort` - This builds on top of the ClusterIP and adds a cluster-wide TCP or UDP port. It makes the Service available outside of the cluster on a stable port.
* `LoadBalancer` - This builds on top of the NodePort and integrates with cloud-based load-balancers

## Creating the service.

To create the Service the manifest must be POSTed to API server.

```
kubectl apply -f svc.yml
```

Which will output the following:

```
service/hello-svc configured
```

## Inspecting Service and Endpoint Objects

### Service

To inspect the deployed service above, the usual `kubectl get` and `kubectl describe` commands can be used:

```
kubectl get svc test-svc
```

Will output the following:

```
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
test-svc   NodePort   100.70.40.2     <none>        8080:30001/TCP  8s
```

Use `kubectl describe` to get the details

```
kubectl describe svc test-svc
```

Which will output the following:

```
Name:                     test-svc
Namespace:                default
Labels:                   <none>
Annotations:              kubectl.kubernetes.io/last-applied-configuration...
Selector:                 app=hello-world
Type:                     NodePort
IP:                       100.70.40.2
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30001/TCP
Endpoints:                100.96.1.10:8080, 100.96.1.11:8080, + more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

### Endpoint

```
kubectl get ep test-svc
```

Which will output the following:

```
NAME        ENDPOINTS                                        AGE
test-svc   100.96.1.10:8080, 100.96.1.11:8080 + 8 more...   1m
```

We can also use `kubectl describe ep` to get more info

```
kubectl describe ep hello-svc
```

Which will output the following:

```
Name:         test-svc
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change...
Subsets:
  Addresses:  100.96.1.10,100.96.1.11,100.96.1.12...
  NotReadyAddresses:   <none>
  Ports:
    Name      Port     Protocol
    ----      ----     --------
    <unset>   8080     TCP
Events: <none>
```

# How Services can be useful in deployments

Services can be used to allow for zero down time deployments.

The way this can be achieved is to make a new Deployment and update one of the version-identifying labels. When this happens there will be two ReplicaSets, new and old. The new ReplicaSet will not replace or delete the old one.

The Service's label selector can be updated to match the new ReplicaSet and all traffic will then be forced to point to the new application with no downtime.

If we want to easily roll back we can update the Service yaml with the previous version which will point back to the old Pods. The old version still existed in this case, but no traffic was being routed to it.

This functionality can be used for all kinds of deploys - blue-greens, canary, etc