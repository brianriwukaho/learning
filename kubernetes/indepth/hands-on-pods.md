# Interacting with Pods

Below is an example manifest describing a singular Kubernetes Pod

```
apiVersion: v1
kind: Pod
metadata:
    name: 
    labels:
        zone: develop
        version: v1
spec:
    containers:
    - name: test-container
      image: test/test:latest
      ports:
      - containerPort: 8000
```

## apiVersion
This field tells you about the API group and API version.

!!! needs more info
## Kind

This field tells Kubernetes the type of object being deployed

## metadata

This section is where you attach a name and labels that help identify and create loose couplings between different objects.

Namespaces that objects should be deployed to can also be defined. If no namespace is defined the Pod will be deployed to the `default` namespace

## spec

This section is where you define the containers that will run in the Pod

# Deploying Pods from a manifest file

To post the manifest to the API server you can do the following in `kubectl`

```
kubectl apply -f pod.yml
```

which should output the following

```
pod/test-pod created
```

the following command can be used to check the status

```
kubectl get pods
```

which will output something similar to 

```
NAME      READY   STATUS             RESTARTS   AGE
test-pod  0/1     ContainerCreating  0          9s
```

You can add a `--watch` flag to `kubectl get pods` to monitor status changes

# Introspecting Running Pods

The following `kubectl get` flags offer more info:
* `-o wide` gives more columns but is a single line of output
* `-o yaml` returns a full copy of the Pod manifest from the cluster store. What it additionally has is a `status` section detailing the current observed state.

Here is an example:
```
kubectl get pods  test-pod -o yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      ...
  name: test-pod
  namespace: default
spec:  #Desired state
  containers:
  - image: test/test:latest
    imagePullPolicy: Always
    name: test-container
    ports:
status:  #Observed state
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2019-11-19T15:24:24Z
    state:
      running:
        startedAt: 2019-11-19T15:26:04Z
...
```
One thing to note is that the Kubernetes Pod objects has more settings than defined in the manifest

## kubectl describe

This command provides a formatted multi-line overview of an object. Here's an example

```
kubectl describe pods hello-pod
```
```
Name:         test-pod
Namespace:    default
Node:         k8s-slave-lgpkjg/10.48.0.35
Start Time:   Tue, 19 Nov 2019 16:24:24 +0100
Labels:       version=v1
              zone=prod
Status:       Running
IP:           10.1.0.21
Containers:
  hello-ctr:
    Image:          test/test:latest
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
Conditions:
  Type           Status
  Initialized    True
  Ready          True
  PodScheduled   True
...
Events:
  Type    Reason     Age   Message
  ----    ------     ----  -------
  Normal  Scheduled  2m   Successfully assigned...
  Normal  Pulling    2m   pulling image "test/test:latest"
  Normal  Pulled     2m   Successfully pulled image
  Normal  Created    2m   Created container
  Normal  Started    2m   Started container
```
## Running Commands in Pods

Another way to inspect a running Pod is to log in or execute commands in it. This can be done with the `kubectl exec` command.

```
kubectl exec test-pod -- ps aux
```
```
PID   USER     TIME   COMMAND
  1   root     0:00   node ./app.js
 11   root     0:00   ps aux
```

Containers in Pods can also be logged-in to via `kubectl exec`

```
kubectl exec -it test-pod -- sh
```
Once inside the container you are free to explore with the usual linux CLI commands

The `-it` flag stands for interactive and allows you to connect the STDDIN and STDOUT of your terminal to the one inside the first container in the pod.

If running a multi-container Pod, you will need to pass the `--container flag` and give the name of the container that you wnt to exec into. 

Container ordering and names can be found using the `kubectl describe pods <pod>` command.

# Kubectl logs
One other useful way to inspect Pods is using the `kubectl logs`. If the `--container` flag is not specified it will execute against the first container in the Pod.

The format of the command is `kubectl logs <pod>` and will show the STDOUT of the container into your terminal.