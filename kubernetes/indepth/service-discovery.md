# Service Discovery

## Service Registration

Service registration is the process of a microservice registering its connection details in a service registry so other microservices can discover it and connect to it

Important things to note are:
1. Kubernetes uses an internal DNS service as its service registry
2. Services, not individual Pods, register with DNS
3. The name, address, and network port of  every Service registered

Kubernetes provides a well known internal DNS Service that is called "cluster DNS". Well known means that it is known to every Pod and container in the cluster.

This DNS is implement in the `kube-system` Namespace as a set of Pods managed by a Deployment called `coredns`. These Pods are fronted by a service called `kube-dns`. Behind the scenes this internal dns is based on a DNS technology called coreDNS (https://coredns.io/) and is run as a Kubernetes-native application.

The following commands can be used to inspect the DNS system

### Service  
```
kubectl get svc -n kube-system -l k8s-app=kube-dns
```
```
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   192.168.200.10   <none>        53/UDP,53/TCP,9153/TCP   3h44m
```
### Deployment
```
kubectl get deploy -n kube-system -l k8s-app=kube-dns
```
```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           3h45m
```
### Pods
```
kubectl get pods -n kube-system -l k8s-app=kube-dns
```
```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-5644d7b6d9-fk4c9   1/1     Running   0          3h45m
coredns-5644d7b6d9-s5zlr   1/1     Running   0          3h45m
```

## Registration with cluster DNS

Every Kubernetes Service is automatically registered with the cluster DNS when it is created. The registration process is as follows:

1. A Service manifest is posted to the API server
2. The request is authenticated, authorised, and and subjected to admission policies
3. The service is allocated a virtual IP address, called a ClusterIP
4. An Endpoints object is created to hold a list of Pods the Service will load balance traffic to
5. The Pod network is configured to handle traffic sent to the ClusterIP
6. The Service's name and IP are registered with the cluster DNS

Any time the cluster DNS observes a new Service object, it creates the DNS records that allow the Service name to be resolved to its ClusterIP. As a result, Applications and Service do not perform service registration as the cluster DNS is constantly scanning to perform this automatically.

```
apiVersion: v1
kind: Service
metadata:
  name: ent <<---- Name registered with cluster DNS
spec:
  selector:
    app: web
  ports:
    ...
```

## The Service back end

The "frontend" of a Kubernetes service is its stable name, IP, and Port.

The "backend" involves involves creating and maintaining a list of Pods that the Service will load-balance traffic to. This is done via the matching labels on Pods with label selectors defined on the Service.

The command below shows the Endpoints object for the Service defined above:

```
kubectl get endpoint ent
```

```
NAME    ENDPOINTS                                    AGE
ent     192.168.129.46:8080,192.168.130.127:8080     14m
```

## Summarising Service Registration

1. POST Service config to API Server
2. Assigned ClusterIP
3. Config persisted to Cluster store
4. Endpoints created with Pod IPs
5. Cluster DNS sees new Service
6. DNS records created
7. Kube proxies pull Service config
8. IPVS rules created

## Service Discovery

For Service discovery to work, every microservice needs to know two things:

1. The name of the remote microservice to connect to
2. How to convert the name to an IP address

The application developer takes care of (1) in code and kubernetes takes care of (2)

Kubernetes automatically configures every container so that it can find and use the cluster DNS to convert Service names to IPs. This is done by populating every container's `/etc/resolve.conf` file with the IP address of a cluster DNS service as well as any search domains that should be appended to unqualified names.

An "unqualified name" is short name such as ent above. Appending a search domain converts an unqualified name into a fully qualified domain name (FQDN) such as `ent.default.svc.cluster.local`

The following snippet is an example `/etc/resolv.conf` that is configured to send DNS queries to the cluster DNS at 192.168.200.10.

```
search svc.cluster.local cluster.local default.svc.cluster.local
nameserver 192.168.200.10
options ndots:5
```

We can see that the `nameserver` field above matches the IP address of the cluster DNS

```
kubectl get svc -n kube-system -l k8s-app=kube-dns
```
```
NAME       TYPE        CLUSTER-IP          PORT(S)                  AGE
kube-dns   ClusterIP   192.168.200.10      53/UDP,53/TCP,9153/TCP   3h53m
```

## Network Magic

Every Kubernetes Node runs a system service called `kube-proxy`. This service is responsible for capturing traffic destined for ClusterIPs and redirecting it to the IP addresses of Pods that match the Service's label selector.

In detail `kube-proxy` is a Pod-based Kubernetes-native app that implements a controller that watches the API server for new Service and Endpoints Objects. When they are found it creates local IPVS that tells the Node to intercept traffic destined for the ClusterIPs and forwards it to individual Pod IPs.

Every time a Node's kernel processes traffic headed for an address on the service network, a trap occurs and the traffic is redirected to the IP of a healthy Pod matching the Service's label selector.

## Summarising Service Discovery

1. Query DNS for Service Name
2. Receive ClusterIP
3. Send traffic to ClusterIP
4. No Route. Send to container's default gateway
5. Forward to Node
6. No Route. Send to Node's default gateway
7. Processed by Node's kernel
8. TRAP (IPVS rule)
9. Rewrite IP destination field to Pod IP

## Service Discovery and Namespacecs

Namespaces partition the address space below the cluster domain. Creating namespaces called `prod` and `dev` will give two address spaces that you can place k8s objects in:
* dev.svc.cluster.local
* prod.svc.cluster.local

Object names must be unique within Namespaces but not across Namespaces. This is useful for parallel development and production namespaces.

Pods in one namespace can connect to Services in that local namespace by using short names. Connecting to objects in a remote Namespace requires FQDNs such as `ent.dev.svc.cluster.local` instead of just `ent`.

In addition to partitioning cluster address space. Namespaces can be good for implementing access control and resource quotes.

Namespaces are not workload isolation boundaries and should not be used as such.

## Namespace example

The YAML below defines two Namespaces, two Deployments, two Services, and a stand-alone jump Pod. The two Services and Deployments have the same name but are deployed to different namespaces.

```
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enterprise
  labels:
    app: enterprise
  namespace: dev
spec:
  selector:
    matchLabels:
      app: enterprise
  replicas: 2
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: enterprise
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - image: nigelpoulton/k8sbook:text-dev
        name: enterprise-ctr
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enterprise
  labels:
    app: enterprise
  namespace: prod
spec:
  selector:
    matchLabels:
      app: enterprise
  replicas: 2
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: enterprise
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - image: nigelpoulton/k8sbook:text-prod
        name: enterprise-ctr
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: ent
  namespace: dev
spec:
  selector:
    app: enterprise
  ports:
    - port: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: ent
  namespace: prod
spec:
  selector:
    app: enterprise
  ports:
    - port: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Pod
metadata:
  name: jump
  namespace: dev
spec:
  terminationGracePeriodSeconds: 5
  containers:
  - name: jump
    image: ubuntu
    tty: true
    stdin: true
```

We then deploy the YAML

```
kubectl apply -f sd-example.yml
```
```
namespace/dev created
namespace/prod created
deployment.apps/enterprise created
deployment.apps/enterprise created
service/ent created
service/ent created
pod/jump-pod created
```

and check that the configuration was correctly applied (The following output is trimmed)

```
kubectl get all -n dev
```

```
$ kubectl get all -n dev
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/ent   ClusterIP   192.168.202.57   <none>        8080/TCP   43s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/enterprise   2/2     2            2           43s
<snip>

$ kubectl get all -n prod
NAME          TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)    AGE
service/ent   ClusterIP   192.168.203.158   <none>        8080/TCP   82s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/enterprise   2/2     2            2           52s
<snip>
```