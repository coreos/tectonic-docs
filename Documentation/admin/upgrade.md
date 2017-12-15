## Upgrading Tectonic &amp; Kubernetes

Use Tectonic Console to control the process by which Tectonic and Kubernetes are updated. Clusters are attached to an update channel and are set to update either automatically, or with admin approval.

<div class="row">
  <div class="col-lg-8 col-lg-offset-2 col-md-10 col-md-offset-1 col-sm-12 col-xs-12 co-m-screenshot">
    <img src="../img/settings-updates.png">
    <div class="co-m-screenshot-caption">Cluster update settings in the Console</div>
  </div>
</div>

During an update, the latest versions of the Tectonic and Kubernetes components are downloaded. A seamless rolling update will occur to install the latest versions. A cluster admin can pause the update at any time.

Please note that the update payload process may affect any or all components in the tectonic-system and kube-system namespaces.

To learn more about how this process is executed, read about [Operators][operators].

## Production and pre-production channels

Tectonic clusters can be conifigured to track either a production or pre-production update channel for the desired "minor" version of Kubernetes, like `1.8` or `1.7`.

| Name | Workload | Timeframe |
|------|----------|-----------|
| Pre-production | Development or testing environments running real workloads, with the goal of catching capability bugs early | Available upon release |
| Production | Clusters serving any amount of production traffic and/or have high reliability and uptime requirements | Promoted 2-4 weeks after initial release |

Configure your desired update channel in the Cluster Settings screen in the Console.

## Upgrading between minor versions of Kubernetes

Before attempting an upgrade to a new "minor" version, ensure you are running the latest Tectonic version of the current minor version. For example, you need to be running the latest `1.7.9-tectonic.3` before upgrading to `1.8.4-tectonic.1`.

To start the opt-in upgrade process, switch your channel to the new minor version, and choose either production or pre-production. After this selection, click Check for Updates to query the new channel for updates.

## Preserve &amp; Restore etcd

If you'd like to preserve and restore etcd data to the new cluster, see the etcd [disaster recovery][etcd-disaster-recovery] guide.


[operators]: https://coreos.com/operators/
[etcd-disaster-recovery]: https://coreos.com/etcd/docs/latest/admin_guide.html#disaster-recovery
