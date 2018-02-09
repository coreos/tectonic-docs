<br>
<div class="alert alert-info" role="alert">
<i class="fa fa-exclamation-triangle"></i><b> Note:</b> This documentation is for an alpha feature. For questions and feedback on the Multi-cluster Alpha program, email <a href="mailto:tectonic-alpha-feedback@coreos.com">tectonic-alpha-feedback@coreos.com</a>.
</div>

# Enabling multi-cluster registry in Tectonic

The Tectonic multi-cluster registry provides a standard way to:
* Discover a list of clusters from anywhere in the multi-cluster environment.
* Centralize management of RBAC policies.
* Centralize management of namespace objects and access to those namespaces.

Together these features allow you to configure base security policies, such as restricting node administration only to cluster admins, on all of your clusters.

Syncing team specific policies, such as creating a namespace for an API deployment in every production region, provides an easy means to enforce standardization.

Tectonic's multi-cluster registry enables overlapping policies, and considers the entire set of objects in the cluster before creating, updating, or deleting any policies. The software will not delete any objects it did not create; only objects created by a Cluster Policy will be deleted.

Note: Workload resources like Deployments, ConfigMaps and Secrets are out of scope for this feature.

**Terminology:**

* **Cluster Policy:** A set of namespaces, role definitions, and role bindings. These are synced to any qualified replica cluster, based on the policy’s label query.
* **Directory cluster:** The central storage location of the cluster policies. This can be any of the clusters included in the multi-cluster registry.
* **Replica cluster:** A cluster that will have policies applied to it, based on labels.

## Prerequisites

* 2 or more Kubernetes 1.7+ Tectonic clusters. (These may be cloud or bare metal clusters.)
* Admin access via RBAC.
* DNS resolution between each replica cluster to the directory cluster.
 (Test with dig <clustername> from a node in the cluster.)
* Ingress allowed from each replica to the directory cluster.

## Installation

Follow the [install guide][install] to configure your clusters.

[install]: install-multi.md
