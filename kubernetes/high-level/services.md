# Services

Pods can fail and fallover, but are replaced when they fail with new IP addresses. Scaling up operations also adds new Pods with new IP addresses. Scaling down also takes these existing Pods away. This results in a lot of IP churn.

Pods are unreliable, so how are applications supposed to rely on them?

The answer is services. Services provide reliable networking for a set of Pods.

A Kubernetes Service is a first class-citizen of the Kubernetes API like Pods and Deployments. Services have a frontend that consts of a stable DNS name IP address and port. The service backend load balances across a dynamic set of Pods. Regardless of Pod scaling behaviour and availability, Services will continue to provide a stable networking endpoint.

tl:dr - A service is a stable network abstraction that provides TCP and UDP load-balancing across a dynamic set of Pods.

Services cannot provide application-layer host and path routing as they are not aware of the application. To do this you must use an Ingress.

# Connecting Pods to Services

Services use labels and label selectors to know which set of Pods to load-balance traffic to.

A Service has a label selector that is a list of all labels a Pod must possess in order for it to receive traffic the Service.

Services only send traffic to healthy Pods that pass health checks.