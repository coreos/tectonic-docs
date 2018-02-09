# Working with Open Cloud Services

Open Cloud Services automate common administrative tasks, such as installation and upgrading, allowing admins to focus on deployments and permissions.

Use Tectonic Console to enable Open Cloud Services for a selected namespace, then initialize and configure the service for use within that namespace.

## Enabling Open Cloud Services

Tectonic admins enable Open Cloud Services on selected namespaces, granting users with access to those namespaces permission to initialize and deploy the service. For more granular control, create custom RBAC roles and permissions to control access to these services within defined namespaces.

Cluster admins may select the teams and namespaces for whom Open Cloud Services will be enabled. For example, admins may allow only certain teams access to Vault, a powerful and secure secret store. Prometheus access may be broader across teams, but limited to namespaces requiring its reporting capacity.

Once enabled in a namespace, normal Kube roles and bindings can be used to further control access to edit or delete these resources.

### Using Tectonic Console

Use Tectonic Console to enable Open Cloud Services for your Tectonic cluster.

1. From *Applications > Open Cloud Catalog*, select an Open Cloud Service, and click *Enable*.
2. In the window that opens, select the namespaces into which the app will be deployed, and click *Enable*.

Once enabled, the Open Cloud Services page will list the version deployed, and the namespaces for which each app is enabled.

### Using kubectl

To enable Open Cloud Services using kubectl, create a `Subscription` resource in the desired namespace.

For example:

```yaml
apiVersion: app.coreos.com/v1alpha1
kind: Subscription-v1
metadata:
  name: etcd
spec:
  channel: alpha
  name: etcd
  source: tectonic-ocs
```

Valid values for `spec.name` are:
* `etc`
* `prometheus`
* `vault`

The Vault OCS will automatically grant the namespace access to its private image repository.

## Creating Instances

Open Cloud Service instances may be deployed into any namespace for which they are enabled.

1. Go to *Applications > Available Applications* and select the namespace for the service you wish to deploy.
2. Click *Create New* to open a YAML manifest template for the instance.
3. Edit the manifest to rename the app, and customize it if necessary, and click *Create*.

A new instance will be deployed into the selected namespace. Once created, Console displays the following information for each instance:

* The *Overview* tab provides detailed information on the app, including version number and the namespace(s) into which the app is deployed. This page also displays graphs which show CPU, Memory, and Filesystem Usage for the app, and the status of its members (Pods).
* The *YAML* tab opens a page in which the custom resource definition may be reviewed for the selected app.
* The *Resources tab* lists and itemizes all related resources for the selected cluster instance, including number of deployed Pods, services, and backups (available options vary, specific to each service).

## Customizing Open Cloud Services

Tectonic Open Cloud Services deploy apps using best practices configuration options to ensure highly available, secure, and fully managed Kubernetes Services. While not recommended, these services may be customized using the configuration options available to individual Operators.

Use Tectonic Console or kubectl to edit the YAML manifest for individual services.

For more information on specific configuration options, see the documentation for the respective operators:

* [etcd operator][etcd-operator]
* [Prometheus operator][prom-operator]
* [Vault operator][vault-operator]

## Updating Open Cloud Services

Trigger a rolling update for Open Cloud Services by clicking down into an instance’s Details page, and editing its YAML manifest.


[etcd-operator]: https://coreos.com/operators/etcd/docs/latest/
[prom-operator]: https://coreos.com/operators/prometheus/docs/latest/
[vault-operator]: https://coreos.com/tectonic/docs/latest/vault-operator/user/vault.html
