# Defining Tectonic user roles

Tectonic Console allows you to define Roles used to grant access to system, namespace or cluster-wide resources.

Each Role is composed of a set of rules, which defines the type of resource and access allowed the Role.

## Creating Roles

From *Administration > Roles*, click *Create Role* to open a YAML template, which may be edited to create a new Role.

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: example
  namespace: default
rules:
  - apiGroups:
      - '' # empty quotes ('') indicate the core API group
    resources:
      - pods
    verbs:
      - get
      - watch
      - list
```

Once the Role is created, click its *Name* in the *Roles* page, then click *Add Rule* to open the *Create Access Rule* page.

From the Create Access Rule page, select:

**Type of Access:** actions that can be taken on Kubernetes resources are grouped into easy categories.
* *Read-only:* allows users to view, but not edit the listed resources.
* *All:* allows full access to all resources, including the ability to edit or delete.
* *Custom:* grants a user-defined set of access privileges, as selected from the complete list of actions.

**Allowed Resources:**
* *Recommended:* grants access to the default set of safe resources, as recommended by Tectonic.
* *All Access:* grants full access to all resources, including administrative resources.
* *Custom:* grants access to a user-defined set of resources, as selected from the *Safe Resources*, *API Resources*, and *API Groups* listed below.
* *Non-resource URLs:* grants access to API URLs that do not correspond to objects.

## Default Roles in Tectonic

Tectonic inherits most of the cluster-wide roles from Kubernetes upstream. The default cluster-wide roles in Tectonic are:

| Cluster Roles | Permissions   |
| ------------- |:-------------|
| cluster-admin | Super-user access. When used in a ClusterRoleBinding, grants full control over every resource in the cluster and in all namespaces. When used in a RoleBinding, grants full control over every resource in the rolebinding's namespace, including the namespace itself.|
| admin         | Full control over all objects in a namespace. Bind this role into a namespace to give administrative control to a user or group. It does not allow write access to resource quota or to the namespace itself.|
| edit          | Read/write access to all common objects, either within a namespace or cluster-wide. Does not allow read/write access to roles or rolebindings. |
| view      | Read-only access to most objects. Can be used cluster-wide, or within a specific namespace. Does not allow viewing roles, rolebindings, or secrets.|

For more information on these roles, see [User-facing Roles][user-facing] in the Kubernetes documentation.

## Assign Users to Roles

To grant users access to Roles, [use a Role Binding][creating-role-bindings].


[role-binding]: creating-accounts.md#creating-role-bindings
[user-facing]: https://kubernetes.io/docs/admin/authorization/rbac/#user-facing-roles
[creating-role-bindings]: creating-accounts.md/#creating-role-bindings
