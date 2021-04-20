# Masters (control plane)

A kubernetes master is a collection of sstem services that make up the control plane of the cluster

The master should be run on high availability in prod

Masters should not run user applications. Their main concern is to manage the cluster

## API Server

Control station of kubernetes. Exposes RESTful API that you can post YAML configuration (manifests) to. These manifests describe the desired state of your app.


## Cluster Store

Persistently stores configuration and state of the cluster. Based on `etcd` and is the single source of truth for the cluster.

Should be run in HA.

## Controller Manager

Background control loops that monitor the cluster and respond to events. Some control loops include, `node controller, endpoint controller, ReplicaSet controller`

Aims to ensure that the current state of the cluster matches desired state

## Scheduler

Watches the API server for new tasks and assigns them to healthy nodes by rank with verious predicate checks. If there are no suitable nodes tasks are marked as scheduled.

## Cloud Controller Manager

Manage integratiosn with underlying cloud vendor technologies and services

# Sumamary

The Control Plane can be though of as the brain of the cluster made up of many small specialised control loops and services.

