# Deployment Theory

Applications get packaged in containers that get wrapped in Pods so they can run on Kubernetes. However, Pod's don't self-heal, scale or allow for easy updates or rollbacks. Deployments do all of these, as a result Pods will almost always be deployed via the Deployment controller.

A single Deployment object can only manage a single Pod template. If you have an application with a 2 Pod templates: backend and frontend, you will need two Deployments.

## Replica Sets

Behind the scenes, Deployments leverage another object called a ReplicaSet. 

At a high level, Deployments use ReplicaSets to provide self-healing and scaling. One can think of Deployments as managing ReplicaSets which manage Pods

## Self-Healing and Scalability

Three concepts are fundamental to everything about Kubernetes:
* Desired state
* Current state
* Declarative model

## Control loops

ReplicaSets implement a background control loop that is constantly checking whether the right number of Pod replicas are present on the cluster. If there aren't enough it adds more, if there are too many it terminates some:

**Kubernetes is constantly making sure that the current state matches the desired state**

The same control loop enables scaling. 

## Rolling Updates

Deployments also enable zero-downtime rolling updates. How? Just update the deployment YAML file and post it to the API server. When we consider the desired vs current state model of Kubernetes, the control loop will seamlessly handle the rest.

How do we go backwards? Just post the previous deployment YAML.

**badabing - badaboom.**

There are also startup probes, readiness probes, and liveness probes that can check the health and status of Pods.

# How to create a deployments

Below is an example Deployment manifest file. 

```
apiVersion: apps/v1  #Older versions of k8s use apps/v1beta1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:latest
        ports:
        - containerPort: 8080
```

## apiVersion

This field specifies the API version. Deployment objects are inthe apps/v1 API group.

## kind

What object we are defining. In this case a Deployment object.

## metadata

Where you Kubernetes objects names and labels

## spec

Anything below `.spec` relates to the Pod. Anything nested below `.spec.template` relates to the Pod template that the Deployment will manage. In this example the Pod template defines a single container.

`.spec.replicas` tells Kubernetes how many Pod replicas to deploy

`.spec.selector` is a list of labels that Pods must have in order for the Deployment to manage them.

`.spec.strategy` tells Kubernetes how to perform updates to the Pods managed by the deployment.

## Accessing Deployments

The normal `kubectl get` and `kubectl describe` commands can be used to see the details of a Deployment.

```
kubectl get deploy test-deploy

```
```
NAME          DESIRED   CURRENT  UP-TO-DATE  AVAILABLE   AGE
hello-deploy  10        10       10          10          24s
```

```
kubectl describe deploy test-deploy
```
```
Name:         hello-deploy
Namespace:    default
Selector:     app=hello-world
Replicas:               10 desired | 10 updated | 10 total ...
StrategyType:           RollingUpdate
MinReadySeconds:        10
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:               app=hello-world
  Containers:
        hello-pod:
          Image:        nigelpoulton/k8sbook:latest
          Port:         8080/TCP
```

Deployments automatically created associated ReplicaSets. 

```
kubectl get rs
```
```
NAME                  DESIRED   CURRENT  READY   AGE
hello-deploy-7bbd...  10        10       10      1m
```

## Perform a rolling Update

Below is an example Deployment that we want to update the image of

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
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: test-pod
        image: test/test:hotfix   # This line changed
        ports:
        - containerPort: 8080
```

### Fields of note

* minReadySeconds - Tells Kubernetes to wait 10 seconds between each Pod being Update. Useful for throttling the rate at which updates occur.
* strategy - Tells Kubernetes that you want the Deployment to:
  * Update using the `RollingUpdate` strategy
  * Never have more than one Pod below the desired state (`maxUnavailable: 1`)
  * Never have more than one Pod above the desired state (`maxSurge: 1`)

We can update the Deployment above by posting the YAML file to the API server

```
kubectl apply -f deploy.yml --record
```
Which will output:
```
deployment.apps/test-deploy configured
```
This updated Deploy may take some time to complete. This is because it will iterate 2 Pods ata time, pulling down the image on each new node, starting the new Pods, and then waiting 10 seconds before moving on to the next two.

The progress of the update can be monitored with `kubectl rollout status`

```
kubectl rollout status deployment test-deploy
````
Which will output:
```
Waiting for rollout to finish: 4 out of 10 new replicas...
Waiting for rollout to finish: 4 out of 10 new replicas...
Waiting for rollout to finish: 5 out of 10 new replicas...
```
You can also get the deploy while the update is in process
```
kubectl get deploy
```
Which will output:
```
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
test-deploy   10        11        5            9           28m
```

## Performing Rollbacks

If you use the `--record` flag on `kubectl apply`, Kubernetes would maintain a documented revision history of the Deployment.

The following command shows the previous Deployment with two revisions.

```
kubectl rollout history deployment test-deploy
```
Which will output:
```
deployment.apps/hello-deploy
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply --filename-deploy.yml --record=true
```
Updating a deployment creates a new ReplicaSet. any previous ReplicaSets are not deleted. This can be verified with `kubectl get rs`

```
kubectl get rs
```

Which will output:

```
NAME                  DESIRED  CURRENT  READY  AGE
hello-deploy-6bc8...  10       10       10     10m
hello-deploy-7bbd...  0        0        0      52m
```