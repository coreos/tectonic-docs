# Role-Based Access Control

Tectonic Identity uses Kubernetes' integrated [Role-Based Access Control][kubernetes-rbac] (RBAC) to manage user roles and permissions within Tectonic clusters.

Use Tectonic Console to define Roles which grant a set of permissions to Accounts through Role Bindings.

By default, Tectonic offers three account types, and two Role and Role Binding types:

Account types:

* User
* Group
* Service Account

Role types:

* Roles: Restrict associated rules to a namespace.
* Cluster Roles: Grant associated Rules across a cluster.

Role Binding types:

* Namespace Role Binding: Defines namespace-specific permissions for users or a group of users.
* Cluster-wide Role Binding: Defines cluster-wide permissions for users or a group of users.

A Role Binding can reference both Roles and Cluster Roles to grant permissions to resources. This allows administrators to define a set of common Roles for the entire cluster, then reuse them within multiple namespaces.

For example, creating a Role Binding in the `dev` namespace that binds a user to the `edit` Cluster Role won't have any impact outside of the `dev` namespace, even though it references a Cluster Role.

An attempt to access a resource or perform a command not allowed by the user's defined permissions will be rejected by the API server.

For more information on User Authentication and access management, see [Creating Accounts][creating-accounts], and [Creating Roles][creating-roles].

## Configuring RBAC

RBAC may be configured using Tectonic Console or kubectl. The following example creates two YAML files to define a Role and a Role Binding for user Jane Doe, which grants her basic access to the cluster.

1. First, create a YAML file which defines the role `support-readonly` that can run commands `get`, `logs`, `list`, and `watch` for namespaces and pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
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

2. Then, bind the role to `jane.doe`'s group `tstgrp`:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
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


[creating-accounts]: creating-accounts.md
[creating-roles]: creating-roles.md
[kubernetes-rbac]: https://kubernetes.io/docs/admin/authorization/rbac/
