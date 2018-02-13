# Adding a Service Account to a Tectonic cluster

Service Accounts are API credentials used by Pods to authenticate with the Kubernetes API. Use a Service Account for any non-human interaction with the cluster.

For example, when an Ingress controller running in a cluster must read Ingress resources, it loads Service Account credentials mounted into the Pod, which are used to authenticate with the API server.

## Mechanics of using a Service Account

These credentials are mounted onto Pods as files, which is an easy format for any programming language to consume. This makes it easy to provide access-controlled identity for apps to talk to other cluster services.

When making API calls from within a Pod, the Service Account's credentials are presented as a bearer token on the API request. Their usernames are derived from their namespace and name: `system:serviceaccount:<namespace>:<name>`.

## Default Service Accounts

Kubernetes automatically creates a `default` service account with relatively limited access in every namespace. If Pods don't explicitly request a Service Account, they are assigned to this `default` account.

Creating additional Service Accounts is recommended. Creation is intended to be lightweight, allowing cluster users to create service accounts for specific tasks (i.e. principle of least privilege).

For more information, see the Kubernetes guides:
* [Managing Service Accounts][manage-service]
* [Configuring Service Accounts for Pods][configure-service]

## Creating a Service Account

Specify a name and namespace for the service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: example
  namespace: default
```

## Assigning Roles

Every Service Account has an associated username that can be granted RBAC Roles, similar to users and groups. Service Accounts are tied to Namespaces, but can be use to access resources in other namespaces.

In a Role Binding, use the *Subject* kind *Service Account*.

```yaml
subjects:
- kind: ServiceAccount
  name: example
  namespace: default
```

Service accounts are configured much like user accounts, in that first a [Role][creating-roles] is created, and then a [Role Binding][creating-accounts] is used to associate the Role with the Service Account.

### Mounting Service Accounts to Pods

Give each application its own Service Account, rather than relying upon the default Service Account. A Service Account credentials can be mounted onto a pod by specifying its name in the pod spec. For example:

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
 name: nginx-deployment
spec:
 replicas: 3
  selector:
    matchLabels:
      k8s-app: nginx
 template:
   metadata:
     labels:
       k8s-app: nginx
   spec:
     containers:
     - name: nginx
       image: nginx:1.7.9
     serviceAccountName: example # note the name of the service account for future reference
```


[creating-accounts]: creating-accounts.md
[creating-roles]: creating-roles.md
[manage-service]: https://kubernetes.io/docs/admin/service-accounts-admin/
[configure-service]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
