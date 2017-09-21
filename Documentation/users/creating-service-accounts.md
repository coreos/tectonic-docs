# Adding a service account to Tectonic cluster

Service accounts are API credentials stored in Kubernetes APIs and mounted onto pods as files, providing access-controlled identity to the services running on pods. In effect, any process running inside a pod uses a service account to authenticate itself to Kubernetes APIs from within a cluster. For example, when an ingress controller running in a cluster must read ingress resources, it loads service account secrets mounted into the pod at known locations to authenticate with the API server. The apps running on the clusters use the service account's secrets as a bearer token. Kubernetes automatically creates a `default` service account with relatively limited access in every namespace. If pods don't explicitly request a service account, they are assigned to this `default` account. However, creating an additional service account is permitted.

Every service account has an associated username that can be granted RBAC roles, similar to other account types. Service accounts are tied to namespaces. Their usernames are derived from their namespace and name: `system:serviceaccount:<namespace>:<name>`. Because RBAC denies all requests unless explicitly allowed, service accounts, and the pods that use them, must be granted access through RBAC rules.

service accounts may be cluster-wide or namespace specific.
You cannot update the service account of an already created pod.
user accounts are for people; service accounts are for processes, which run in pods.
service accounts
Service account creation is intended to be more lightweight, allowing cluster users to create service accounts for specific tasks (i.e. principle of least privilege).

Cluster Admin guide to Managing Service Accounts:https://kubernetes.io/docs/admin/service-accounts-admin/

User Guide to Configure Service Accounts for Pods: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/

## Creating a service account

To create a service account use `kubectl create` command or Tectonic Console.


To use Tectonic Console to create Service Accounts, follow the directions to [Create Role Bindings][creating-accounts.md#creating-role-bindings] for a Tectonic user account.

## Granting access rights to service account

Access rights are granted to a service account associated with a role by using a Role Binding. Do either of the following in Tectonic Console:

* Use the *Role Bindings* option under *Administration*.  Create a *Role Binding*, then select a default Role. For example: `admin`.
* Use the *Roles* option under *Administration*. Create a new role by using the YAML editor. Then use the *Role Bindings* option to create a type of Role Binding and bind to the new role.

### Using Tectonic Console

#### Granting a Cluster-wide role to service account

i need to better understand the differences between how cluster-wide vs. namespace service accounts are defined and work. the information here and in the UI is contradictory. - beth

----------------- check if we want to keep this next bit or not. it was pulled from the doc that was called 'user-management.md' ---------------------

### Using kubectl

In this example, a Cluster Role Binding, `etcd-rolebinding`, is created for the `etcd-operator` role  using `kubectl`. This role will have read access over the ingress resources in the `tectonic-system` namespace.

1. Create a YAML file `etcd-operator.yaml` to define the role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: etcd-operator
rules:
- apiGroups:
  - etcd.coreos.com
  resources:
  - clusters
  verbs:
  - "*"
- apiGroups:
  - extensions
  resources:
  - thirdpartyresources
  verbs:
  - "*"
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  verbs:
  - "*"
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - endpoints
  - persistentvolumeclaims
  - events
  verbs:
  - "*"
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - "*"
```

2. Create a second YAML file, `etcdoperator.yaml`, to define a Cluster Role Binding which gives administrative privileges to the service account within the `tectonic-system` namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
 name: example-etcd-operator
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: admin
subjects:
- kind: ServiceAccount
 name: example etcd operator
 namespace: tectonic-system
```
3. Use `kubectl create` to create the service account:

```
kubectl create -f serviceaccount etcdoperator.yaml
```

If creating the service account is successful, the following message is displayed:

```
serviceaccount "example-etcd-operator" created
```

4. Verify once again by fetching the service accounts:

```
kubectl get serviceaccounts
```

Locate `example-etcd-operator` in the list of service accounts:

```
   NAME                     SECRETS    AGE
   default                     1          1d
   example-etcd-operator       1          5m
   .....
 ```

Give each application its own service account, rather than relying upon the default service account. A service account can be mounted onto a pod by specifying its name in the pod spec. For example:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
 name: nginx-deployment
spec:
 replicas: 3
 template:
   metadata:
     labels:
       k8s-app: nginx
   spec:
     containers:
     - name: nginx
       image: nginx:1.7.9
     serviceAccountName: public-ingress # note the name of the service account for future reference
```


[user-management]: user-management.md
[ldap-user-management]: ldap-user-management.md
[saml-user-management]: saml-user-management.md
[identity-management]: identity-management.md#default-roles-in-tectonic
