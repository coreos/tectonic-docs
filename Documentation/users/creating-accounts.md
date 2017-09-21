# Creating Tectonic accounts

Use Tectonic Console to grant access rights to users using Role Bindings.

First, create Roles. Then, associate users with defined Roles and cluster access using Role Bindings.

Tectonic allows you to create user, group, and admin accounts for your clusters. Any of these three account types may be granted access to a cluster, or to a specific namespace within a cluster.

Tectonic allows you to create both cluster wide and namespace specific user accounts.

Grant access rights to users by creating Role Bindings. Role Bindings associate access permissions (cluster wide, or namespace specific) with [Roles][https://coreos.com/tectonic/docs/latest/admin/identity-management.html#default-roles-in-tectonic] and [Subjects][link to somewhere subjects are defined - user, group, or service account. (no admin subject?)]

When removed from Tectonic Identity authentication (LDAP or SAML integration), users and groups are cached, and no longer granted the permissions created through Role Bindings.

Use Role Bindings to create the following three user types:

(note - i want to verify these definitions. they're inconsistent, so i'm not sure they are correct)

* **Users:** have access to all common objects within a cluster, but do not have access to change RBAC policies.
* **Groups:** grant defined access to multiple users. Logged in (authenticated) users are grouped according to their RBAC group memberships.

Permissions granted an individual user are an aggregate of permissions for all roles assigned the user account, and all permissions granted the roles for all groups to which the user belongs.

Individual users may be part of multiple groups.

* **Admins:** have full control over all resources within their cluster or namespace.

##  Prerequisites

User authentication must be enabled before defining user roles. Use Tectonic Identity to integrate with your existing authentication system. For more information, see:

* [Static user management][user-management]
* [LDAP user management][ldap-user-management]
* [SAML user management][saml-user-management]

## Creating Role Bindings

1. In Tectonic Console, go to *Administration > Role Bindings* and click *Create Binding* to open the *Create Role Binding* page.

2. Select the access level for the Role Binding:
 * *Namespace Role Bindings* grant access only within the listed namespace.
 * *Cluster-wide Role Bindings* grant access across the Tectonic cluster.
 While a Cluster Role can be bound down the hierarchy to a Namespace Role Binding, a Namespace Role can't be promoted up the hierarchy to be bound to a Cluster Role Binding.
 Namespace users and groups have read-access to Pods. Admins have full control over all resources. (is that true? read-only? - beth)

3. Enter a *Name* for the Role Binding. This name will be used to xxx.

4. Select the Namespace to which the Role Binding grants access (if applicable).

4. Select a *Role Name* for the binding.
Roles are defined using the *Administration > Roles* page in Tectonic.
For more information, see [Default Roles in Tectonic][identity-management].
should be - for more info, see Creating Roles - beth
Each role is made up of a set of rules, which defines the type of access and resources that are allowed to be manipulated.

5. Select a *Subject* for the Role Binding:
 * *User* grants permissions to a single user. Users must preexist in the console (right? )
 * *Group* grants permissions to a defined group. For more information, see [creating groups in tectonic][xx].
 * *Service Account* creates a service account, for use with xxx. For more information, see xxx

6. Enter the *Subject Name* for the binding. The Name must exist in the Tectonic system, and may be one of the following three values (dependent on the Subject selected for the Binding):
(User accounts are intended to be global. Names must be unique across all namespaces of a cluster, future user resource will not be namespaced. Service accounts are namespaced.)
* For a User, enter an authenticated email address for a user
* For a Group, enter a Group Name (as defined where?),
* For a Service Account, enter a service account name (as defined where?)

7. Click *Create Binding* to create the binding, and open the *Role Bindings* page.


## Using kubectl to create ClusterRoleBindings

`ClusterRoles` grant access to types of objects in any namespace in the cluster. Tectonic comes preloaded with three `ClusterRoles`:

1. user
2. readonly
3. admin

`ClusterRoles` are applied to a `User`, `Group` or `ServiceAccount` through a `ClusterRoleBinding`. A `ClusterRoleBinding` can be used to grant permissions to users in all namespaces across the entire cluster, where as a `RoleBinding` is used to grant namespace specific permissions. The following `ClusterRoleBinding` resource definition grants an existing user the admin role.

First, create a YAML file called `admin-test.yaml` with the following content:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: admin-test
# subjects holds references to the objects the role applies to.
subjects:
  # May be "User", "Group" or "ServiceAccount".
  - kind: User
    # Preexisting user's email
    name: test1@example.com
# roleRef contains information about the role being used.
# It can only reference a ClusterRole in the global namespace.
roleRef:
  kind: ClusterRole
  # name of an existing ClusterRole, either "readonly", "user", "admin",
  # or a custom defined role.
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

Then, use kubectl to apply the `ClusterRoleBinding` RBAC resource definition:

```
kubectl create -f admin-test.yaml
```

The new `ClusterRoleBinding` can viewed in Tectonic Console under the Administration tab.

The `ClusterRoleBinding` may be deleted to revoke users' permissions.

```
kubectl delete -f admin-test.yaml
```

For more information see the [Kubernetes RBAC documentation][k8s-rbac].


----------------  fodder ----------------------


### Creating a Cluster user

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-12 col-xs-offset-1">
    <a href="../img/cluster-user.png" class="co-m-screenshot">
      <img src="../img/cluster-user.png">
    </a>
  </div>
</div>



[user-management]: user-management.md
[ldap-user-management]: ldap-user-management.md
[saml-user-management]: saml-user-management.md
[identity-management]: identity-management.md#default-roles-in-tectonic
