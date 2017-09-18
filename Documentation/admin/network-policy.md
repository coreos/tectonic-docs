# Configuring network policies using Calico

This document describes declaring network policies to manage communication between pods. Tectonic uses Calico to implement network policies in Kubernetes clusters. Network policies in conjunction with a network provider, such as Calico, secure applications running on Kubernetes clusters.

## About network policies

Kubernetes by default allows intra-cluster communication. No rules pre-exist to regulate the traffic between pods in a cluster. Kubernetes uses [NetworkPolicy][network-policy-resource] resources to label pods and define rules that specify how traffic is controlled between selected groups of pods. These policies allow to declare which namespaces are allowed to communicate, what protocol is used, and which port numbers to enforce each policy on. Applying a network policy to an open connection would terminate the connection immediately. Network policies in effect create firewalls between pods to secure microservices running on each pod.

## About Calico plugin

Calico plugin provides namespace isolation at the network layer, and policy-driven network security to Kubernetes pods. The network policy created in Kubernetes is automatically maps to a Calico network policy and the rules defined in the manifest are enforced on the cluster.

## Enabling Calico in a Tectonic installation

To enable Calico as the network provider, set `tectonic_calico_network_policy: true` in the `terraform.tfvars` during installation.

## Configuring network policies

This section provides a few example use cases and sample manifests for implementing network policies.

## Prerequisites

* Tectonic Cluster is up and running

  For information on getting Tectonic up and running, see [Deploying an application on Tectonic][deploy-first-app].

* Calico plugin is enabled on Tectonic

* User credentials are configured

  See [Configuring credentials][configure-credentials].

* `kubectl` is configured to interact with the cluster

  See [Configuring credentials][first-app].

* An example application is deployed and exposed it via a service.

  This document uses [Kubernetes Guestbook application][deploy-app] for demonstration.

### About the Guestbook application

The Guestbook application writes its data to a Redis master instance and reads data from the Redis slave instance. The application has a web frontend serving the HTTP requests written in PHP. It is configured to connect to the redis-master service for write requests and the redis-slave service for read requests. The redis-slave and redis-master services are only accessible within the cluster because the default type for a service is `ClusterIP`. `ClusterIP` provides a single IP address for the set of pods the service is pointing to. This IP address is accessible only within the cluster.
A frontend service is configured to be externally visible, so a client can request the service from outside the cluster. The service is exposed through `LoadBalancer`. The Guestbook deployment has one pod for redis-master, two pods for redis-slave, and three pods for the frontend.

### Limit the traffic to Guestbook within a namespace

Using the example manifest, create a network policy that restricts access to the redis-master service. The rule defined in the policy allows connection only from the pods (microservices) with the label `k8s-app: guestbook`.

#### Use cases: allow inbound traffic from selected pods

* Prevent any other pods communicating with a selected pod.

* Isolate traffic to a service from other pods.

* Restrict connections to a database only to the applications using it.

#### Creating the network policy

1. Save the following manifest to `allow-redis.yaml`:

  ```YAML
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-redis
    namespace: default
  spec:
    podSelector:
      matchLabels:
        role: db
    ingress:
      - from:
        - podSelector:
            matchLabels:
              k8s-app: guestbook
  ```

2. Run the following:

  `kubectl apply -f allow-redis.yaml`

  If creating the network policy is successful, the following message is displayed:

  `networkpolicy "allow-redis" created`

#### Testing the connection

Test whether the network policy is blocking the traffic. To do that, run a pod without the `k8s-app: guestbook` label:

```sh
kubectl run busybox --rm -ti --image=busybox /bin/sh
```

Attempt to access the redis-master service from a pod without the correct labels will time out:

```sh
Waiting for pod default/busybox-754571723-t0m33 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 redis-master
Connecting to redis-master (10.3.146.192:80)
wget: download timed out
/ #

```

### Allowing traffic from an external service

#### Use cases

To test this scenario, create an ingress controller in the cluster. Use the following YAML to create the ingress.

```yaml

kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: simple
  namespace: team-doc
  annotations:
    kubernetes.io/ingress.class: tectonic
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: sample.dev.se.k8s.work
      http:
        paths:
          - path: /
            backend:
              serviceName: frontend
              servicePort: 80

```




Cookie shop

The ngnix deployment created on the cluster runs three pods in the default namespace on the To see how Kubernetes network policy works, create an nginx deployment and expose it via a service. This runs two nginx pods in the default namespace, and exposes them through a service called nginx.

You should be able to access the new nginx service from other pods. To test, access the service from another pod in the default namespace. Make sure you haven’t enabled isolation on the namespace.
Start a busybox container, and use wget on the nginx service:


[deploy-first-app]: first-app.md#deploying-an-application-on-Tectonic
[configure-credentials]: first-app.md#configuring-credentials
[deploy-app]: second-app.md
[network-policy-resource]: https://kubernetes.io/docs/api-reference/v1.7/#networkpolicy-v1-networking
