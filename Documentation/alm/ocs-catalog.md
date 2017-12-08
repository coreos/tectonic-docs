# Tectonic Open Cloud Services Catalog

Open Cloud Services (OCS) are software services made available to Tectonic users on demand and in their own environment. OCSs take care of the heavy lifting of maintaining open source projects by automating maintenance tasks such as regular, one-click, zero-downtime updates, disaster recovery, and horizontal scaling, much like public cloud services. Unlike other public cloud services like AWS DynamoDB, Tectonic Open Cloud Services are first class Kubernetes resources and are truly portable to any datacenter or cloud. Because these Open Cloud Services run in your environment, they also allow you to see the container, logs, flags, and config file within your Kubernetes environment.

Open Cloud Services provide application inventory and continuous delivery. They allow infra-admins to easily deploy services (Vault,  etcd, or Prometheus) into the namespace of their choice, and app developers to easily create and manage the services’ instances.

## Available Open Cloud Services

Tectonic provides the following Open Cloud Services:

* [etcd OCS][etcd-ocs] provides a one-click, fully managed etcd key-value store for use by any application on-top of a Tectonic cluster, and automates tasks related to operating an etcd cluster.
  etcd OCS is Highly Available, and provides safe upgrades and backups automation.
* [Prometheus OCS][prom-ocs] provides a one-click, fully managed, application monitoring and metrics stack for both operations and applications teams on-top of a Tectonic cluster. The Prometheus OCS can also transform these metrics into action using Prometheus's Alertmanager to route, email, page, or message teams when something goes wrong with an application or the container infrastructure.
Prometheus OCS is Highly Available, Kubernetes native, and high performance.
* [Vault OCS][vault-ocs] provides a one-click, fully managed Vault encryption service and secrets management tool that can store existing secrets, or dynamically create new secrets on top of a Tectonic cluster. When enabled in Tectonic, an etcd cluster will be automatically created to manage the Vault instance.
  Vault OCS is secure by default, with automated creation of TLS certificates between all components. Vault OCS is also Highly Available, and provides safe upgrades.

[etcd-ocs]: etcd-ocs.md
[prom-ocs]: prom-ocs.md
[vault-ocs]: vault-ocs.md
