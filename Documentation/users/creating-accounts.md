# Creating Tectonic accounts

This guide covers using the Tectonic Console to grant cluster-wide or namespace-specific access for human users. For programmatic access for your software, see [Creating Service Accounts][creating-service-accounts].

First, create [Roles][creating-roles] which define rules of access. Then, associate users with Roles using "Role Bindings".

## Users vs Groups

Role Bindings can be used to associate a single user with a Role, or a group of users with a Role.

* **Users:** match the email address (static users) or username (LDAP, SAML) used to log in.
* **Groups:** match one or more group names from LDAP or SAML backends.

Permissions granted an individual user are an aggregate of permissions for all roles assigned the user account, and all permissions granted the roles for all groups to which the user belongs.

Individual users may be part of multiple groups.

##  Prerequisites

User authentication must be enabled before defining user roles. Use Tectonic Identity to integrate with your existing authentication system. For more information, see:

* [Service accounts][service-accounts]
* [LDAP integration][ldap-integration]
* [SAML integration][saml-integration]

## Creating Role Bindings

In Tectonic Console, go to *Administration > Role Bindings* and click *Create Binding* to open the *Create Role Binding* page.

1. Select the access level for the Role Binding:
 * *Namespace Role Bindings* grant access only within the listed namespace.
 * *Cluster-wide Role Bindings* grant access across the Tectonic cluster.

2. Enter a *Name* for the Role Binding. Meaningful names are easy for others to audit.

3. If creating a Namespace Role Binding, select the Namespace to which the Role Binding grants access.

4. Select a *Role Name* for the binding.
The Role must preexist in the Tectonic cluster. Use the *Administration > Roles* page to define Roles. For more information, see [Defining Tectonic user roles][creating-roles].

5. Select the type of *Subject* for the Role Binding:
 * *User* grants permissions to a single user. Users must exist in the authentication system configured with [Tectonic Identity][tectonic-identity-overview].
 * *Group* grants permissions to a defined group.
 * *Service Account* creates a service account, to allow software to use the Kubernetes API. For more information, see [Adding a service account to a Tectonic cluster][creating-service-accounts].

6. Enter the *Subject Name* for the binding. The Name must exist in the Tectonic system, and may be one of the following three values (dependent on the Subject selected for the Binding):
 * For a User, enter an email address (static users) or username (LDAP, SAML).
 * For a Group, enter a Group Name from LDAP or SAML backends.
 * For a Service Account, enter a service account name that exists in the cluster.

7. Click *Create Binding* to create the binding, and open the *Role Bindings* page.

## Using kubectl to create ClusterRoleBindings

`ClusterRoles` grant access to objects in any namespace in the cluster. Tectonic offers four default `ClusterRoles`:

* cluster-admin: Super-user access. When used in a ClusterRoleBinding, grants full control over every resource in the cluster and in all namespaces. When used in a RoleBinding, grants full control over every resource in the rolebinding's namespace, including the namespace itself.
* admin: Admin access within a namespace.
* edit: Read/write access to most objects in a namespace. It does not allow read/write access to roles or rolebindings.
* view: Read-only access to most objects in a namespace.

For more information on these roles, see [Default Roles in Tectonic][default-roles] [User-facing Roles][user-facing] in the Kubernetes documentation.


`ClusterRoles` are applied to a `User`, `Group` or `ServiceAccount` through a `ClusterRoleBinding`. A `ClusterRoleBinding` can be used to grant permissions to users in all namespaces across the entire cluster, whereas a `RoleBinding` is used to grant namespace specific permissions.

The following `ClusterRoleBinding` resource definition grants an existing user the admin role.

First, create a YAML file called `admin-test.yaml` with the following content:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin-test
subjects: # holds references to the objects the role applies to.
  - kind: User # May be "User", "Group" or "ServiceAccount".
    name: test1@example.com # Preexisting user's email
roleRef: # contains information about the role being used.
         # It can only reference a ClusterRole in the global namespace.
  kind: ClusterRole
  # name of an existing ClusterRole, either "readonly", "user", "admin",
  # or a custom defined role.
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

Then, use kubectl to apply the `ClusterRoleBinding` RBAC resource definition, and create the account:

```
$ kubectl create -f admin-test.yaml
clusterrolebinding "admin-test" created
```

In Tectonic Console, go to *Administration > Role Bindings* to view the new `ClusterRoleBinding`.

The `ClusterRoleBinding` may be deleted to revoke users' permissions.

```
$ kubectl delete -f admin-test.yaml
```

## Removing Access

Changes to a Role or Role Binding will take place immediately. Revoking access to a user may not be fully complete until the user's session token expires. The default expiration time is 24 hours.

## More Info

For more information see the [Kubernetes RBAC documentation][k8s-rbac].


[creating-service-accounts]: creating-service-accounts.md
[creating-roles]: creating-roles.md
[k8s-rbac]: https://kubernetes.io/docs/admin/authorization/rbac/
[ldap-integration]: ldap-integration.md
[saml-integration]: saml-integration.md
[tectonic-identity-overview]: tectonic-identity-overview.md
[service-accounts]: creating-service-accounts.md
[ldap-integration]: ldap-integration.md
[saml-integration]: saml-integration.md
[user-facing]: https://kubernetes.io/docs/admin/authorization/rbac/#user-facing-roles
[default-roles]: creating-roles.md#default-roles-in-tectonic
