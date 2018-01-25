# etcd Open Cloud Service

Tectonic’s etcd Open Cloud Service provides a one-click, fully managed etcd key-value store for use by any application on-top of a Tectonic cluster. The service creates and maintains a set of resources that describe etcd clusters, allowing admins to easily deploy and manage etcd clusters in any namespace

* **Easy highly available setup:** Multiple instances of etcd are clustered together and secured. Individual failures or networking issues are transparently handled to keep the cluster up and running.
* **Safe Upgrades:** Rolling out a new etcd version is as easy as updating the etcd cluster definition; everything is automatically handled with a safe rolling update to the new version.
* **Backups included:** Trigger backups to an object store as part of disaster recovery planning.

## Deploying etcd OCS

Use Tectonic Console to enable the etcd OCS for selected namespaces. By default, the etcd Open Cloud Service will deploy a 3 member cluster, which may be resized or updated by editing its YAML manifest.

For more information on enabling the etcd OCS and creating instances, see [Working with Open Cloud Services][using-ocs].

Objects created using the etcd OCS will be labeled `app=etcd` and `etcd_cluster=<cluster-name>`.

Using the etcd Open Cloud Service to deploy an etcd cluster will create the following Kubernetes objects:
* An etcd Custom Resource Definition (CRD)
* 3 Pod etcd cluster
* 2 Services, one for internal communication and one for use with etcdctl/API

## Working with Kubernetes Services

For every etcd cluster created Tectonic will create an etcd client service in the same namespace with the name `<cluster-name>-client`.

The client service is of type `ClusterIP` and is accessible only from within the Kubernetes cluster's network.

To expose this address outside of the cluster, create a new service of type LoadBalancer or NodePort.

For more information on accessing this service, see [Client service][client-service] in the etcd-operator documentation set.


[client-service]: https://github.com/coreos/etcd-operator/blob/master/doc/user/client_service.md
[using-ocs]: using-ocs.md
