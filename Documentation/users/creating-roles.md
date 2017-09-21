# Defining Tectonic user roles

Tectonic Console allows you to define Roles used to grant access to system, namespace or cluster-wide resources.

Each Role is composed of a set of rules, which defines the type of access and resources that are allowed to be manipulated.

## Creating Roles

Click *Create Role* to open a YAML template, which may be edited to create a new Role.

Once the Role is created, click on its *Name* in the *Roles* page, then click *Add Rule* to open the *Create Access Rule* page.

<Create Access Rule page>

From the Create Access Rule page, select:

**Type of Access:**
* *Read-only:* allows users to view, but not edit the listed resources.
* *All:* allows full access to all resources, including the ability to edit or delete.
* *Custom:* grants a user-defined set of access privileges, as selected from the Actions listed below. (are these kubectl commands? - beth)

**Allowed Resources:**
* *Recommended:* grants access to the default set of safe resources, as recommended by Tectonic.
* *All Access:* grants full access to all resources, including administrative resources.
* *Custom:* grants access to a user-defined set of Resources, as selected from the Safe Resources, API Resources, and API Groups listed below.
* *Non-resource URLs:* grants access to API URLs that do not correspond to objects.
(what’s the basis for the list of selectable ‘Safe Resources’? All tectonic created Kubernetes resource types? - beth)

## Default Roles in Tectonic

Tectonic inherits most of the roles from Kubernetes upstream. There are cluster-wide roles, namespace roles, and system roles. ingress-controller is a cluster-wide role to handle Tectonic ingress traffic.
(why was ingress-controller mentioned?)

The default Cluster-wide roles in Tectonic are:

| Cluster Roles | Permissions   |
| ------------- |:-------------|
| cluster-admin | Full control over all the objects in a cluster.|
| admin         | Full control over all objects in a namespace. Bind this role into a namespace to give administrative control to a user or group.|
| user          | Access to all common objects, either within a namespace or cluster-wide. |
| edit          | Access to all common objects, either within a namespace or cluster-wide, but is prevented from changing the RBAC policies. |
| view      | Read only view for all objects. Can be used cluster-wide, or just within a specific namespace.|

(i'll want a list of ALL default roles - beth)
