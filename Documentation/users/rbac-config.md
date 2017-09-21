# Role-Based Access Control

Tectonic Identity uses Kubernetes' integrated Role-Based Access Control (RBAC) to manage user roles and permissions within Tectonic clusters.

Role-based access for users may be defined using Tectonic Console.

For more information on User Authentication and access management, see
[Creating Accounts][creating-accounts], and
[Creating Roles][creating-roles]

For more information on Kubernetes RBAC authorization API, see [Using RBAC Authorization][kubernetes-rbac].

## Configuring RBAC

RBAC may be configured using Tectonic Console or kubectl. The following example creates two YAML files to define a Role and a Role Binding for user Jane Doe.

### An example configuration

The following shows an example of granting user `jane.doe@example.org` basic access to the cluster.

1. Add a role named `support-readonly` that can run commands `get`, `logs`, `list`, and `watch` for namespaces and pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: support-readonly
rules:
- apiGroups:
  - ""
  attributeRestrictions: null
  resources:
  - namespaces
  - namespaces/finalize
  - namespaces/status
  - pods
  verbs:
  - get
  - logs
  - list
  - watch
```

2. Bind the role to `jane.doe`'s group `tstgrp`:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: support-reader
  namespace: kube-system
subjects:
  - kind: Group
    name: tstgrp
roleRef:
  kind: ClusterRole
  name: support-readonly
  apiGroup: rbac.authorization.k8s.io
```

Tectonic Console and `kubectl` now reflect the updated role and binding:

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-12 col-xs-offset-1">
    <a href="../img/ui-permission-granted.png" class="co-m-screenshot">
      <img src="../img/ui-permission-granted.png">
    </a>
  </div>
</div>

3. Verify all pods are up and running:

```bash
$ kubectl --kubeconfig=janeDoeConfig --namespace=tectonic-system get pods

NAME                                         READY     STATUS    RESTARTS   AGE
default-http-backend-4080621718-f3gql        1/1       Running   0          2h
kube-version-operator-2694564828-crpz4       1/1       Running   0          2h
...
```

### Unauthorized access

An attempt to access a resource or perform a command to which a user does not have access will be rejected by the API server:

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-12 col-xs-offset-1">
    <a href="../img/ui-permission-notallowed.png" class="co-m-screenshot">
      <img src="../img/ui-permission-notallowed.png">
    </a>
  </div>
</div>

```bash
$ kubectl --kubeconfig=janeDoeConfig --namespace=tectonic-system get services

Error from server (Forbidden): the server does not allow access to the requested resource (get services)
```


[creating-accounts]: creating-accounts.md
[creating-roles]: creating-roles.md
[kubernetes-rbac]: https://kubernetes.io/docs/admin/authorization/rbac/
